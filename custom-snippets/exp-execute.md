::: {.cell .markdown}
### Execute the experiment and visualize the results
:::


::: {.cell .markdown}

Next, we will generate some TCP flows between the two end hosts, and use it to observe the behavior of the TCP congestion control algorithm.

On the "juliet" host, we will receive the TCP flows. On the "romeo" host, we will send three TCP flows using TCP Reno, and monitor the congestion window and slow start threshold. It will run for about one minutes.

:::


::: {.cell .code}
```python
import time
# start the receiver on juliet
_ = juliet_ssh.execute('iperf3 -s -1  -D')
# on romeo, start monitoring the connection
_ = romeo_ssh.execute_thread('bash ss-output.sh 10.10.2.100')
time.sleep(1)
# then, on romeo, start sending TCP flows
_ = romeo_ssh.execute('iperf3 -c juliet -P 3 -i 10 -t 60 -C reno')
```
:::

::: {.cell .markdown}

Once it is done, we will stop the "monitor" and it will save its output to a file -

:::

::: {.cell .code}
```python
_ = romeo_ssh.execute("kill -INT -$(ps x | grep '[b]ash ss-output.sh' | awk '{print $1}')")
```
:::


::: {.cell .markdown}

We will transfer the file from the FABRIC resource:

:::

::: {.cell .code}
```python
import os
romeo_ssh.download_file(os.path.join(os.getcwd() + "/sender-ss.csv"), "sender-ss.csv")
```
:::

::: {.cell .code}
```python
import pandas as pd
import matplotlib.pyplot as plt

df = pd.read_csv("sender-ss.csv", names=['time', 'sender', 'retx_unacked', 'retx_cum', 'cwnd', 'ssthresh'])

# exclude the "control" flow
s = df.groupby('sender').size()
df_filtered = df[df.groupby("sender")['sender'].transform('size') > 100]

senders = df_filtered.sender.unique()

time_min = df_filtered.time.min()
cwnd_max = 1.1*df_filtered.cwnd.max()
dfs = [df_filtered[df_filtered.sender==senders[i]] for i in range(3)]

fig, axs = plt.subplots(len(senders), sharex=True, figsize=(12,8))
fig.suptitle('CWND over time')
for i in range(len(senders)):
    if i==len(senders)-1:
        axs[i].plot(dfs[i]['time']-time_min, dfs[i]['cwnd'], label="cwnd")
        axs[i].plot(dfs[i]['time']-time_min, dfs[i]['ssthresh'], label="ssthresh")
        axs[i].set_ylim([0,cwnd_max])
        axs[i].set_xlabel("Time (s)");
    else:
        axs[i].plot(dfs[i]['time']-time_min, dfs[i]['cwnd'])
        axs[i].plot(dfs[i]['time']-time_min, dfs[i]['ssthresh'])
        axs[i].set_ylim([0,cwnd_max])


plt.tight_layout();
fig.legend(loc='upper right', ncol=2);
```
:::

::: {.cell .markdown}

At the beginning of each flow, it operates in slow start mode, where the congestion window increases exponentially. When a congestion event occurs, as indicated by the receipt of multiple duplicate ACKs, the slow start threshold is set to half of the current CWND, and then the CWND is reduces to the slow start threshold.

We'll often see packet losses occur at the same time in multiple flows sharing a bottleneck (as in the figure above), because when the buffer is full, new packets arriving from all flows are dropped. 

:::


