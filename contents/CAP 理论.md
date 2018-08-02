# CAP 理论

通常情况下，一开始业务需求比较简单直观，用户量不大，采用单机或少量集群即可完成对业务的技术支撑，而随着业务系统不断复杂以及用户规模不断状态，对性要求也会随之增大，单纯提升单机性能从长远来看肯定是不现实的，只有不断横向拓展集群规模，加强对于系统服务的解耦管理，才能有效应对业务复杂度与系统吞吐量的增加。而对于集群管理之下的分布式系统而来，势必会带来CAP理论问题。

CAP理论：

|-|-|
|---|---|
|Consistency||
|Availability||
|Partition Tolerance||


![](http://robertgreiner.com/uploads/images/2014/CAP-overview.png)


![](http://robertgreiner.com/uploads/images/2014/CAP-CP.png)


![](http://robertgreiner.com/uploads/images/2014/CAP-AP.png)