在 zookeeper 的集群中，各个节点共有下面 3 种角色和 4 种状态：
角色：leader、follower、observer

leader：负责数据的写/读
follower：负责数据的写/读
observer：负责数据的读，不参与数据的写和半数写入投票

状态：leading、following、observing、looking

LOOKING：当前 Server 不知道 leader 是谁，正在搜寻。
LEADING：当前 Server 即为选举出来的 leader。
FOLLOWING：leader 已经选举出来，当前 Server 与之同步。
OBSERVING：observer 的行为在大多数情况下与 follower 完全一致，但是他们不参加选举和投票，而仅仅接受(observing)选举和投票的结果

ZXID：事务 id，是一个 64 位的数字，高位表示 epotch，每有一次 leader 选择该值进行+1 操作。低位用于每个事务期间的计数

leader 选举（myid,zxid）

启动阶段

1. 将自己的的（myid,zxid）在集群中进行投票信息发送
2. 集群中的其他机器在收到投票信息后进行判断该投票是否有效，如是否为当批次投票，投票的机器是否为 LOOKING 状态
3. 将其他机器的票和本地自己的票进行对比。优先对比 ZXID，如果 ZXID 相同则对比 MYID，值大的票优先成为 Leader。
   对于其他票比自己的票优先更高，修改本地的票信息。并且将新的投票信息发送到集群中，如果本地的票优先级等于或大于接受到的票则不过任何处理。投票结束完成后进行统计，判断是否已经有过半机器接受到相同的投票信息。有过半的机器则认为已经选举出 Leader。
   一旦确定了 Leader，每个服务器就会更新自己的状态，如果是 Follower，那么就变更为 FOLLOWING，如果是 Leader，就变更为 LEADING。

Leader 失效阶段：
选举步骤同启动阶段

节点的监听

get -w 本节点监听
ls -w 字节点监听
state -w 属性监听

监听是一次性，收到个更新
需要重新创建监听

权限

A admin 权限
C 创建权限
W 写入权限
R 读取权限
D 删除权限

Schema

WORLD 全世界
Ip ip 段
Auth 账号密码
Digest 密文账号

数据节点类型

永久节点，
永久序列节点，
临时节点（无字节点）
临时序号节点（无字节点）
