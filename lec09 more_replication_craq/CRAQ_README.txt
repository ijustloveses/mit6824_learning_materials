## Introduction

### ��ͳ�� chain replication method

- ������Ҫ replicate ���ݵĽڵ���֯Ϊһ�� chain
- д�����ӵ�һ���ڵ㿪ʼ��ÿһ���ڵ�д���Ժ󣬴�����һ���ڵ㣬ֱ�����һ���ڵ�Ҳд�룬��Ϊ committed
- Ȼ��д��ɹ��Ļظ������һ���ڵ�������ǰ���ͣ�ֱ����һ���ڵ���յ��ɹ�д��Ļظ��󣬷��ظ��ͻ���
  ��Ȼ��Ҳ������ tail ֱ�ӻظ����ͻ��ˣ�����һЩ
- �����������һ���ڵ�����ֻ�����нڵ㶼���ºú�Ż᷵�������ݣ����򷵻�������

�ŵ� (vs raft)��
- �ͻ��˵Ķ�д����ֿ��� head �� tail �ֱ������� raft �Ļ���raft leader Ҫ�����������󣬸�������
- CR �� head �ڵ�ֻ������һ������������д���󣬶� raft ҪȺ��
- CR �Ķ�����ֻ���漰��һ������������ raft �Ķ�����ͬ��Ҫ�� majority���ʴ��漰����ķ�����
- Failure Recovery ���� Raft ���򵥣����ǻ��� CRAQ �㷨�н���

��Ҫ�����⣺
- ���������������һ���ڵ��ϣ������������һ���ڵ��Ϊ hotspot��ѹ������

### CRAQ - Chain Replication with Apportioned Queries

- ����ǿһ���Ե�ͬʱ���ṩ�������ĵ��ӳٺ͸����¡�������Ҫ�Ƕ���ϵͳ����������ƥ��ֻ��֤����һ���Ե�ϵͳ
- ������ apportioned queries: �Ѷ������ֲ������нڵ���
- �����Ƕ��������ܻ� stale����ȡ����ʱ�����ݣ�CRAQ ֧���������� staleness acceptence ����
- �ײ�ʹ���� ZK(ZooKeeper) ��Ϊ configuration manager����Ȼԭ����Ҳ������ raft��paxos ��
- һЩ���߼����Ż����ڽ��е���
  + mini-transaction ֧�֣���������� Atomically ִ��
  + multicast ֧�֣��� large-object update ��д���Ż�
  
CRAQ v.s. Raft or ZooKeeper
- Raft��ǿһ���ԣ�����ֻ�ܴ� leader ����д
- ZooKeeper����Э�˶������һ���ԣ���ȡ���������нڵ�����ȡ
- CRAQ��ǿһ���ԣ����ҿ��������нڵ�������ȡ
- CRAQ �� Raft or ZooKeeper ���ŵ�

����ȥ CRAQ �� Raft �� ZooKeeper ��Ҫ�ð���������ô�����ǣ�Ҳ��ȱ��
- Raft & ZooKeeper ��ֻҪ majority quorum ��ɹ�ʶ�Ϳ�����
- CRAQ Ҫ���нڵ㶼��ɹ�ʶ������ֻҪ��һ���ڵ����ˣ�����д�������Ӧ�ͻᱻ������ľͰЧӦ
- ��һ���ڵ�ҵ����ڱ� ZK �жϳ������߳���Ⱥ֮ǰ��д����ֻ�� hold �ȴ�


## System Model

### Object-based Storage ϵͳ�Ĳ���ԭ���һ��ģ��

����ԭ��
- write(objID, V) 
- read(objID) -> V

consistency model
- Strong Consistency�����������Ƕ�ȡ�����µ�д��
- Eventual Consistency�����ܶ�ȡ�� stale������������ replicas ��д���������ݣ�֮���ٻ��ȡ staled ����

### ��ͳ�� chain replication 

- ������ Introduction ������ʾ���Ͳ�׸���ˡ���Ȼ����ǿһ���ԣ�committed ֮�����һ���ڵ�һ��������������

### CRAQ

����д����ԭ���ʵ��
- ÿ���ڵ㶼�������ݵĶ���汾������ÿ�� object���汾���ǵ��������ģ�ÿ���汾����һ�� clean/dirty ��ʶ
- ��д�������� chain ���ݵ�ĳ���ڵ㣬�ýڵ���յ���д�� object ��һ���°汾����������°汾����汾 list
  + ����ڵ㲻�� tail (���һ���ڵ�)���汾��ʶΪ dirty��Ȼ����д����������һ���ڵ㴫��
  + ����� tail����ʶΪ clean���ð汾����Ϊ committed������ tail �� committed ��Ϣ��ǰ��Ľڵ㴫��
- �� committed ��Ϣ��ǰ��Ľڵ���գ��ýڵ�ͰѶ�Ӧ�汾��ʶΪ clean������֮ǰ�İ汾����ȫɾ����
- ���� chain �е��κ�һ���ڵ㴦���������������������˶�����ĳ�������
- ���ڵ���յ�һ����ȡ����
  + ����汾����ߵ����°汾��ʶΪ clean����ôֱ�ӷ�������汾������
  + ������ tail �ڵ㷢������ѯ�� tail �ڵ㵱ǰ���� committed �İ汾�ţ�Ȼ�󷵻�����汾�ŵ�����
    * ��ʹѯ�ʵ����°汾�ź󡢷��ؿͻ���֮ǰ��tail �ڵ������� committed �˸��µİ汾���Ⲣ����Υ��һ����

���ܷ���
- Read-Mostly Workloads�������ڵ㶼�� clean �ģ�����ֱ�ӷ��ؽ�����ʴ˼���������˶����������
- Write-Heavy Workloads��
  + �����ڵ㶼�� dirty �ģ��ʴ˶�Ҫ��ѯ tail �ڵ㣬��ѯ���µ� committed �汾������ tail ���Ϊ hotspot
  + ���������ѯ�������Ը�������ɣ��ʴ�������Ȼ�ȵ��㴦�������Ĵ�ͳ chain replication ����
  + �����������Խ�һ�������� tail������ֻ������ѯ���󣬶������������

CRAQ ֧���������͵�һ����ģ��
- Strong Consistency���������������ܵ����̣���֤��ǿһ����
- Eventual Consistency
  + ��Щϵͳ������ staled ���ݣ��Ի�ȡ����(���������͵��ӳ�)����ô���ǿ��Է������� tail ��ѯ���߼�
  + ÿ���ڵ�ֱ�ӷ��ظýڵ������� committed �汾�����ݣ���ʹ�и��µ� dirty �汾����Ҳ����ȥ tail ѯ��
  + �������ͻ��˴Ӳ�ͬ�ڵ���ȡ��ͬ�����ݣ����ܻ�õ���ͬ�汾�Ľ������Щ����� staled �ģ����ǲ��õ�
  + �õĵط��ǣ��ͻ�����Զ��ĳһ���ڵ��ȡĳ�����ݵĻ���������ݵİ汾һ�����������ģ����յõ����½��
- Eventual Consistency with Maximum-Bounded Inconsistency
  + ������������෴�����᷵�ؽڵ����°汾�����ݣ���������汾�� clean �Ļ��� dirty ��
  + ������Ҫ��һ�����ƣ���ν maximum inconsistency period ���ڵ��������ݣ�ͨ����ʱ����߰汾������

Failure Recovery in CRAQ
- ǰ�� 1���ͻ��˺͸���������֮��������Լ������������໥�����󣬶�����ͨ�� Response ��֪�������Ƿ�ɹ�
- ǰ�� 2��CRAQ ��ÿ���ڵ���Ҫ֪������ chain �ϵ�ǰ����ڵ㣨predecessor and successor���Լ� Head and Tail
- ��� Head failed����ô�� successor �ͻ�ȡ����֮��committed write һ�����ᶪ����;�� write Ҳ���ᶪ
  + ������ Head �մ�����ĳ���󣬻�û�����¸��ڵ�͹��ˣ���ʱ�ͻ���һ�������յ��ظ���ֻҪ resend ����
- ��� Tail failed�������ˣ��� predecessor ȡ����֮���κ� write �����ᶪ
  + �������µ� Tail �ڵ���Ҫ�ж�һ�£��ǲ�����ĳ�� write �Լ��Ѿ�ȷ�ϣ����ǻ�δ����ȷ�ϣ����Ի���ȷ��֮
- �м�ڵ���������滻����ɾ���߼��ͺ�˫��������ͬ�������߼���ȫһ��
  + ��ʱ���滻�ڵ�֮ǰ�Ľڵ���Ҫ�ط� recent writes (����ȫ�� dirty write) ��֮��Ľڵ�

Split-Brain��
- һ������������Ľڵ�����ղ��� Head ��������ô֪�� Head �ǲ��ǹҵ��ˣ����ȡ����֮���� Head ���ڣ�
- ����ýڵ��Լ��жϽڵ��״̬�ǲ����ʵģ����������������ף������������� split-brain ���������
- ʵ�ʵ�������ʹ�� ZooKeeper ��Ϊ���ù��������� higher level ����������Ⱥ�Ľڵ�״̬

���ܵ��Ż���
- Mini-Transaction on CRAQ
  + ϣ��֧�ֶ�������ԭ���Բ�����ȫ���ɹ�����ȫ��ʧ��
  + ���������������ͬһ�� Key ��Ҳ����ͬһ�� object����ʱ�Ǻ� trivial to implement
    * ���� Prepend/Append�� Incr/Decr�� Test-and-set�������Ķ����������Ϊһ�� batch ����
	* ������д������������ᱻӰ�죬��� head ��Ҫ�ȴ� transaction �е�ĳЩ keys ��� clean ״̬
  + ������������ͬһ�� chain ��
    * Sinfonia ʵ����һ�� mini-transaction �㷨��ʹ���� optimistic 2-phase commit Э��
	* ������д������������ᱻӰ�죬��� head ��Ҫ�ȴ� transaction �е�ĳЩ keys ��� clean ״̬
  + ����������ֲ��ڲ�ͬ�� chains ��
    * ͬ���� optimistic 2-phase commit Э��ʵ�֣�ֻҪ�ڸ��� chains �� heads ����ס transaction ��ص� keys
- Lowering Write Latency with Multicast
  + д�����ʱ�򣬲�������ʽ����������һ���� Multi-cast �������ڵ�
  + ֻ��һС���� metadata message ��Ҫͨ����ʽ�������� commit ȷ�Ͽ����� Tail Multi-cast �����нڵ�
  

## ZooKeeper Coordination Service in CRAQ

### ���� 
- tracking �ͼ�� group membership�����ĸ��ڵ�ҵ��ˣ��Ա���� Failure Recovery
- ZooKeeper �� watch ���Ʊ�֤�� CRAQ �ڵ���ڼ�Ⱥ��Ӻ�ɾ���ڵ�ʱ�յ�֪ͨ
- store meta-data��ͬ���� meta-data �����仯ʱ������ע�Ľڵ�Ҳ���յ�֪ͨ

ZooKeeper ͨ��û�ж����ڶ� DataCenters ʱ����������Ż�
- ZooKeeper ��һ��ʵ���У�����֪�� DataCenters Topology & Hierarchy������Ϣ�����ڵ��� WAN �ϴ���
- CRAQ ��ʵ���У��� DC �� ZooKeeper traffic��ÿ�� DC ά���Լ��� local ZooKeeper
- ��֤�ڵ������յ� local ZK nodes ��֪ͨ���������Ľڵ㣬ֻ��������ص� global ZK nodes ��֪ͨ

### ʵ��

Datacenter �еĽڵ���Ϣ
- һ�� CRAQ �ڵ�ᴴ��һ�� Ephemeral file in /nodes/dc_name/node_id���ļ�����Ϊ ip port ��Ϣ
- ͬʱ�� watch /nodes/dc_name�������ͱ�֤�˵���� DC ���ӻ��߼����˷������ڵ�ʱ�������нڵ㶼���յ�֪ͨ

chain ��Ϣ
- ��һ���ڵ��յ�����Ҫ����һ�� chain ʱ������һ���ļ��� /chains/chain_id
- ���ļ��������� CRAQ chain �Ĳ��������������ע������ֻ�� chain ��������Ϣ���������� chain ��ǰ�Ľڵ��б�
- ����Ҫ���뵽��� chain �Ľڵ㣬���� watch ����ļ��������Ϳ����յ� metadata �����仯��֪ͨ















