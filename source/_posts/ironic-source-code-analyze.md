---
title: 关于ironic的一些源码理解
date: 2023-10-06 22:35:10
tags:
- ironic
categories:
- openstack
---

Ironic是一个OpenStack项目，它提供裸机（而不是虚拟机）。它可以独立使用，也可以作为OpenStack云的一部分使用，并与OpenStack keystone、nova、neutron、glance和swift服务集成。
<!--more-->
# 关键技术

pxe

dhcp

nbp

tftp

ipmi

# ironic 介绍

ironic api

ironic conductor

ironic inspect

# 裸金属节点

**添加节点**

裸金属为物理服务器，通过资源类属性唯一标识，可在placement库resource_provider表中查看到。添加节点后只会注册资源类信息，没有cpu、内存信息，资源类格式为CUSTOME_XXX

**设置节点状态**

裸金属节点分别有enroll，manageable，available等，[官网的状态机设计](https://docs.openstack.org/ironic/latest/_images/states.png)，可通过ironic manage/inspector/provide更改状态，裸机自检前需要将节点置为manageable状态，创建实例前需要将节点置为available状态

**节点inspect**

ironic inspect通过ironic-python-agent搜集节点信息，包括cpu、内存、架构等。在飞腾环境下，无法执行inspect相关操作、需要手动添加节点的上述信息。

注：手动根据节点网卡的mac地址创建port，若步骤3已完成ironic inspect则无需执行此步骤

*TODO*：通过创建虚拟机的流程创建裸金属实例时，节点缺少cpu等信息会报错，错误原因需要补充

# 裸金属实例

## 发起创建

**创建flavor**

在创建裸金属实例前，需要调整用户所属项目的配额，因为物理服务器cpu、内存配置较高，默认配额资源可能不够，从而导致报错
创建模板、设置模板的cpu、内存值为0，同时设置资源类，否则节点调度会失败，原因见添加节点

**nova api**

通过horizon或其他定制的web可视化页面，向nova api发起创建裸金属实例的请求

## 资源申明

nova api通过rabbitmq远程调用nova conductor，nova condutor创建实例对象同时调用nova schduler筛选可用的compute node，主要通过资源类确定可用的计算节点，然后RPC调用nova compute的`build_and_run_instance()`

注：nova-comute和nova-compute-ironic的区别在于，非容器化部署时只有nova-comute且需用户在conf修改driver为ironic，容器化部署提供了名为nova-compute-ironic的容器镜像，无需修改conf

**nova compute**

nova-compute-ironic服务首先声明资源占用，修改resource tracker的缓存，使用build生成实例所需的资源

```python
# 创建vif并挂载至node对应port
        try:
            LOG.debug('Start building networks asynchronously for instance.',
                      instance=instance)
            network_info = self._build_networks_for_instance(context, instance,
                    requested_networks, security_groups)
            resources['network_info'] = network_info
# 根据bdm创建volume并挂载
            block_device_info = self._prep_block_device(context, instance,
                    block_device_mapping)
            resources['block_device_info'] = block_device_info
```
ironic driver阶段执行以下步骤

```python
# 更新裸金属节点的信息
        node = self._get_node(node_uuid)
        flavor = instance.flavor

        self._add_instance_info_to_node(node, instance, image_meta, flavor,
                                        block_device_info=block_device_info)
# 创建裸金属所需的逻辑卷，如deploy image和instance image（需要打印日志，此处为猜测）
        try:
            self._add_volume_target_info(context, instance, block_device_info)
        except Exception:
            with excutils.save_and_reraise_exception():
                LOG.error("Error preparing deploy for instance "
                          "on baremetal node %(node)s.",
                          {'node': node_uuid},
                          instance=instance)
                self._cleanup_deploy(node, instance, network_info)
# 校验裸金属节点的deploy、storage、power接口，
        validate_chk = self.ironicclient.call("node.validate", node_uuid)
# 生成configdrive
        configdrive_value = None
        if configdrive.required_by(instance):
            extra_md = {}
            if admin_password:
                extra_md['admin_pass'] = admin_password

            try:
                configdrive_value = self._generate_configdrive(
                    context, instance, node, network_info, extra_md=extra_md,
                    files=injected_files)
# 请求ironic api 接口set_provision_state为active
        try:
            self.ironicclient.call("node.set_provision_state", node_uuid,
                                   ironic_states.ACTIVE,
                                   configdrive=configdrive_value)
```

**ironic api**

ironic api接收`provision_action`请求，通过rabbitmq远程调用ironic conductor

```python
    def _do_provision_action(self, rpc_node, target, configdrive=None,
                             clean_steps=None, rescue_password=None):
        topic = api.request.rpcapi.get_topic_for(rpc_node)
        # Note that there is a race condition. The node state(s) could change
        # by the time the RPC call is made and the TaskManager manager gets a
        # lock.
        if target in (ir_states.ACTIVE, ir_states.REBUILD):
            rebuild = (target == ir_states.REBUILD)
            api.request.rpcapi.do_node_deploy(context=api.request.context,
                                              node_id=rpc_node.uuid,
                                              rebuild=rebuild,
                                              configdrive=configdrive,
                                              topic=topic)
```

## 部署准备

ironic conductor使用task manager管理异步任务，检查节点是否处于维护或rebuild，然后开始部署

```python
with task_manager.acquire(context, node_id, shared=False,
                          purpose='node deployment') as task:
    deployments.validate_node(task, event=event)
    deployments.start_deploy(task, self, configdrive, event=event)
```

### start_deploy

ironic/conductor/deployment.py中会进行以下检查

```python
# 确认实例镜像是否为whole_disk_image
def start_deploy(task, manager, configdrive=None, event='deploy'):

    node = task.node

    if event == 'rebuild':
        ...省略...

    iwdi = images.is_whole_disk_image(task.context, node.instance_info)
    driver_internal_info = node.driver_internal_info
    driver_internal_info['is_whole_disk_image'] = iwdi
    node.driver_internal_info = driver_internal_info
    node.save()
    
# Validate driver_info for ipmitool driver和Validate the deployment information for the task's node
    try:
        task.driver.power.validate(task)
        task.driver.deploy.validate(task)
        utils.validate_instance_info_traits(task.node)
        conductor_steps.validate_deploy_templates(task, skip_missing=True)
        
# 开始进行部署
    try:
        task.process_event(
            event,
            callback=manager._spawn_worker,
            call_args=(do_node_deploy, task,
                       manager.conductor.id, configdrive),
            err_handler=utils.provisioning_error_handler)
```

### do_node_deploy

ironic/conductor/deployment.py中deploy流程如下

```python
def do_node_deploy(task, conductor_id=None, configdrive=None):
    ...省略...
# 判断configdrive    
    try:
        if configdrive:
            if isinstance(configdrive, dict):
                configdrive = utils.build_configdrive(node, configdrive)
            _store_configdrive(node, configdrive)
            
# deploy前准备
    try:
        task.driver.deploy.prepare(task)
        
# deploy的步骤更细
    try:
        # This gets the deploy steps (if any) and puts them in the node's
        # driver_internal_info['deploy_steps']. In-band steps are skipped since
        # we know that an agent is not running yet.
        conductor_steps.set_node_deployment_steps(task, skip_missing=True)
       
    ...省略...
    do_next_deploy_step(task, 0, conductor_id)
```

deploy interface配置为iscsi时，在ironic/drivers/modules/iscsi_deploy.py中，

```python
# 进行pxe引导的准备工作
        deploy_utils.populate_storage_driver_internal_info(task)
        if node.provision_state in [states.ACTIVE, states.ADOPTING]:
            task.driver.boot.prepare_instance(task)
```

boot interface配置为pxe，在ironic/drivers/modules/pxe_base.py中

```python
# 缓存kernel和ramdisk镜像
            if task.driver.storage.should_write_image(task):
                # Make sure that the instance kernel/ramdisk is cached.
                # This is for the takeover scenario for active nodes.
                instance_image_info = pxe_utils.get_instance_image_info(
                    task, ipxe_enabled=self.ipxe_enabled)
                pxe_utils.cache_ramdisk_kernel(task, instance_image_info,
                                               ipxe_enabled=self.ipxe_enabled)
             
# 生成neutron dhcp中tftp的配置，调用neutron接口更新neutron dhcp port
            dhcp_opts = pxe_utils.dhcp_options_for_instance(
                task, ipxe_enabled=self.ipxe_enabled, ip_version=4)
            dhcp_opts += pxe_utils.dhcp_options_for_instance(
                task, ipxe_enabled=self.ipxe_enabled, ip_version=6)
            provider = dhcp_factory.DHCPFactory()
            provider.update_dhcp(task, dhcp_opts)
            
# 生成pxe流程中bootfile的cfg
                pxe_utils.build_service_pxe_config(
                    task, instance_image_info, root_uuid_or_disk_id,
                    ipxe_enabled=self.ipxe_enabled)
                boot_device = boot_devices.PXE
```

在ironic/conductor/steps.py中，关于这一步骤没有仔细了解，大概是更新了node的deploy_steps信息等

```python
def set_node_deployment_steps(task, reset_current=True, skip_missing=False):
    """Set up the node with deployment step information for deploying.

    Get the deploy steps from the driver.

    :param reset_current: Whether to reset the current step to the first one.
    :raises: InstanceDeployFailure if there was a problem getting the
             deployment steps.
    """
    node = task.node
    driver_internal_info = node.driver_internal_info
    driver_internal_info['deploy_steps'] = _get_all_deployment_steps(
        task, skip_missing=skip_missing)
    if reset_current:
        node.deploy_step = {}
        driver_internal_info['deploy_step_index'] = None
    node.driver_internal_info = driver_internal_info
    node.save()
```

### do_next_deploy_step

在`ironic/conductor/deployment.py`中，`do_next_deploy_step()`方法根据上述配置好的deploy step循环执行

```python
def do_next_deploy_step(task, step_index, conductor_id):
    ...省略...

    for ind, step in enumerate(steps):
        # Save which step we're about to start so we can restart
        # if necessary
        node.deploy_step = step
        driver_internal_info = node.driver_internal_info
        driver_internal_info['deploy_step_index'] = step_index + ind
        node.driver_internal_info = driver_internal_info
        node.save()
        interface = getattr(task.driver, step.get('interface'))
        LOG.info('Executing %(step)s on node %(node)s',
                 {'step': step, 'node': node.uuid})
        try:
            result = interface.execute_deploy_step(task, step)
```

## 部署

源码走到这一步已经完成了创建裸金属实例的所有准备，下面只需要重启节点进入pxe引导完成系统安装

### iscsi deploy

在上述的`do_next_deploy_step()`中，笔者猜测先执行的step为`iscsi_deploy.py`中`class ISCSIDeploy()`类的deploy。在`deploy()`方法中，if语句的执行猜测为第一个分支
`deploy()`实际没有进行有意义的部署，只从glance下载裸金属实例需要用到的系统镜像，然后调用`continue_deploy()`

*TODO*：上述猜测均需要实际环境中打印日志验证

```python
    def deploy(self, task):
        node = task.node
        if manager_utils.is_fast_track(task):
            LOG.debug('Performing a fast track deployment for %(node)s.',
                      {'node': task.node.uuid})
            deploy_utils.cache_instance_image(task.context, node)
            check_image_size(task)
            # Update the database for the API and the task tracking resumes
            # the state machine state going from DEPLOYWAIT -> DEPLOYING
            task.process_event('wait')
            self.continue_deploy(task)
```

`continue_deploy()`方法调用`do_agent_iscsi_deploy()`使用iSCSI完成裸金属系统的安装，调用`prepare_instance_to_boot()`完成

```python
    def continue_deploy(self, task):
        task.process_event('resume')
        node = task.node
        LOG.debug('Continuing the deployment on node %s', node.uuid)

        uuid_dict_returned = do_agent_iscsi_deploy(task, self._client)
        root_uuid = uuid_dict_returned.get('root uuid')
        efi_sys_uuid = uuid_dict_returned.get('efi system partition uuid')
        prep_boot_part_uuid = uuid_dict_returned.get(
            'PrEP Boot partition uuid')
        self.prepare_instance_to_boot(task, root_uuid, efi_sys_uuid,
                                      prep_boot_part_uuid=prep_boot_part_uuid)
        self.reboot_and_finish_deploy(task)
```

**do_agent_iscsi_deploy**

`do_agent_iscsi_deploy()`方法首先设置iSCSI存储服务的目标，然后在`continue_deploy()`函数中进行iscsi连接节点的系统盘

```python
def do_agent_iscsi_deploy(task, agent_client):
    node = task.node
    i_info = deploy_utils.parse_instance_info(node)
    wipe_disk_metadata = not i_info['preserve_ephemeral']

    # 设置iSCSI存储服务的目标
    iqn = 'iqn.2008-10.org.openstack:%s' % node.uuid
    portal_port = CONF.iscsi.portal_port
    conv_flags = CONF.iscsi.conv_flags
    result = agent_client.start_iscsi_target(
        node, iqn,
        portal_port,
        wipe_disk_metadata=wipe_disk_metadata)

    # 在ironic端完成
    address = urlparse.urlparse(node.driver_internal_info['agent_url'])
    address = address.hostname

    uuid_dict_returned = continue_deploy(task, iqn=iqn, address=address,
                                         conv_flags=conv_flags)
```

在`iscsi_deploy.py`文件中存在`continue_deploy()`同名的函数和类方法，不能搞混。`continue_deploy()`函数先根据用户镜像的类别选择调用`deploy_disk_image()`和`deploy_partition_image()`，最后会清理镜像缓存

```python
def continue_deploy(task, **kwargs):
    
    ...省略...
    
    try:
        if node.driver_internal_info['is_whole_disk_image']:
            uuid_dict_returned = deploy_disk_image(**params)
        else:
            uuid_dict_returned = deploy_partition_image(**params)
    
    ...省略...
    
    deploy_utils.destroy_images(node.uuid)
```

`deploy_disk_image()`函数会调用`populate_image()`函数，本质是通过dd命令将缓存的用户镜像拷贝至节点系统盘

```python
def deploy_disk_image(address, port, iqn, lun,
                      image_path, node_uuid, configdrive=None,
                      conv_flags=None):
    with _iscsi_setup_and_handle_errors(address, port, iqn,
                                        lun) as dev:
        disk_utils.populate_image(image_path, dev, conv_flags=conv_flags)

        if configdrive:
            disk_utils.create_config_drive_partition(node_uuid, dev,
                                                     configdrive)

        disk_identifier = disk_utils.get_disk_identifier(dev)

    return {'disk identifier': disk_identifier}
```

**prepare_instance_to_boot**


```python
    def prepare_instance_to_boot(self, task, root_uuid, efi_sys_uuid,
                                 prep_boot_part_uuid=None):
        
        ...省略...
        
        node = task.node
        if deploy_utils.get_boot_option(node) == "local":
            # Install the boot loader
            self.configure_local_boot(
                task, root_uuid=root_uuid,
                efi_system_part_uuid=efi_sys_uuid,
                prep_boot_part_uuid=prep_boot_part_uuid)
        try:
            task.driver.boot.prepare_instance(task)
```

此处的源码与笔者理解的思路有些冲突，待查看实际环境的日志后再做进一步解析

```
def prepare_instance(self, task):

```

