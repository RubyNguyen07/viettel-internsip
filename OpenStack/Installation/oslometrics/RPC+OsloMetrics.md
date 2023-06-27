## RPC 

Remote Procedure Call (RPC) is a powerful technique for constructing distributed, client-server based applications. It is based on extending the conventional local procedure calling so that the called procedure need not exist in the same address space as the calling procedure. The two processes may be on the same system, or they may be on different systems with a network connecting them. 

Disadvantages: 
- Remote procedure calling (and return) time (i.e., overheads) can be significantly lower than a local procedure.
- This mechanism is highly vulnerable to failure as it involves a communication system, another machine, and another process.

## Oslo.metrics

This Oslo metrics API supports collecting metrics data from other Oslo libraries and exposing the metrics data to monitoring system. (e.g. as Prometheus format). 

Enables operator to monitor usage of oslo libraries: 
- Number of RPC calls 
- Number of RPC exceptions 
- Time used to process RPC calls 

[Diagram](https://github.com/thanh474/internship-2020/blob/master/ThanhBC/Openstack/osloMetrics/osloimg/oslo-2.png)

## Resources
https://speakerdeck.com/line_developers/oslo-dot-metrics-monitoring-openstack-rpc-calls?slide=13
 