::: {.cell .markdown}
### Additional configuration of resources for this experiment
:::

::: {.cell .code}
```python
# get an SSH session on each host and router
romeo_ssh  = slice.get_node(name="romeo")
juliet_ssh = slice.get_node(name="juliet")
router_ssh = slice.get_node(name="router")
```
:::


::: {.cell .code}
```python
# configure setting on romeo
_ = romeo_ssh.execute('sudo sysctl -w net.ipv4.tcp_no_metrics_save=1')
```
:::

::: {.cell .markdown}

Now, we will configure the "router" node. Run the following cell to configure it as a 1 Mbps bottleneck, with a buffer size of 0.1 MB, in both directions:

:::

::: {.cell .code}
```python
# configure setting on router
for iface_obj in [slice.get_interface(i) for i in ["router-net1-p1", "router-net2-p1"]]:
    iface = iface_obj.get_device_name()
    cmds = f"""
    sudo tc qdisc del dev {iface} root
    sudo tc qdisc add dev {iface} root handle 1: htb default 3
    sudo tc class add dev {iface} parent 1: classid 1:3 htb rate 1Mbit
    sudo tc qdisc add dev {iface} parent 1:3 handle 3: bfifo limit 0.1MB
    """
    _ = router_ssh.execute(cmds)
```
:::


::: {.cell .markdown}

The parameters of the TCP congestion control algorithm, such as congestion window and slow start threshold, are tracked at the sender side. To visualize these, we will use a script that checks socket statistics and saves them to a file at regular intervals. The next cell uploads this script to the "romeo" host.

:::

::: {.cell .code}
```python
import os
script_path = os.path.join(os.getcwd() + "/scripts", "ss-output.sh")
romeo_ssh.upload_file(script_path, "ss-output.sh")
```
:::
