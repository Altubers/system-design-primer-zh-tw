# 在 AWS 上設計一個可以擴展到百萬使用者的系統

*注意：本文件某些連結直接連到 [系統設計主題的索引](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E7%B3%BB%E7%B5%B1%E8%A8%AD%E8%A8%88%E4%B8%BB%E9%A1%8C%E7%9A%84%E7%B4%A2%E5%BC%95)來避免重複的內容。你可以參考連結來取得相關重點、設計的取捨和選擇。*

## 步驟一：描述使用情境與限制

> 蒐集問題的需求、資訊和範圍。
> 透過詢問問題來瞭解使用情境和限制。
> 討論你的假設。

在這裡沒有面試者會幫你釐清上面的問題，所以我們會預先定義一些使用情境和限制。

### 使用情境

解決這個問題需要透過不斷來回修正的方式：1) **負載壓力測試**、2) **描述瓶頸**、3) 解決瓶頸，並提出替代方案、4) 重複以上步驟。上面這幾個步驟可以很好的幫助你解決可擴展系統架構的基本問題。

除非你本來就對 AWS 有相關背景知識，或是你打算申請的職位需要 AWS 的知識，否則對於 AWS 的細部知識並非必要。然而，**絕大多數在這裡提到的準則，你都可以用在其他非 AWS 的環境上**。

#### 我們將所要解決的問題限縮在以下範圍

* **使用者**會產生讀取和寫入的請求
    * **服務**本身會處理請求、儲存使用者資料並且回傳結果
* **服務**本身需要從服務少量的使用者一路擴展到百萬等級的使用者
    * 我們會討論一個通用性的原則來讓架構本身可以處理大量的使用者請求
* **服務**本身具備高可用 (High-Availability)

### 限制與假設

#### 狀態假設

* 流量並非均勻的分佈
* 資料為關連式
* 從 1 個使用者擴展到數千萬名使用者
    * 我們用以下表示方式來代表使用者不斷增加：
        * Users+
        * Users++
        * Users+++
        * ...
    * 1000 萬名使用者
    * 每月 10 億次寫入
    * 每月 1000 億次讀取
    * 讀寫比例為 100:1
    * 每次寫入大小為 1 KB

#### 計算使用量

**向你的面試人員詢問你是否可以用比較粗略的方式來計算使用量**

* 每月 1 TB 的新資料
    * 每次寫入 1 KB * 每月寫入 10 億次
    * 三年共 36 TB 的新資料
    * 假設大部分的寫入都是新資料，而不是更新既有的資料
* 平均每秒 400 次寫入
* 平均每秒 40,000 次寫入

一些筆記：

* 每月 250 萬秒
* 每秒 1 次請求 = 每月 250 萬次請求
* 每秒 40 次請求 = 每月 1 億次請求
* 每秒 400 次請求 = 每月 10 億 次請求

## 步驟二：進行高階設計

> 提出所有重要元件的高階設計

![Imgur](http://i.imgur.com/B8LDKD7.png)

## 步驟三：設計核心元件

> 深入每個核心元件的細節

### 使用情境：使用者發送一個讀取或寫入請求

#### 目標

* 當只有 1 到 2 個使用者時，你只需要基本的架構
    * 只需要一台機器
    * 需要擴展時使用垂直擴展
    * 做好監控來發現瓶頸

#### 從一台機器開始

* **網頁伺服器** 在 EC2
    * 儲存使用者資料
    * [**MySQL 資料庫**](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E9%97%9C%E9%80%A3%E5%BC%8F%E8%B3%87%E6%96%99%E5%BA%AB%E7%AE%A1%E7%90%86%E7%B3%BB%E7%B5%B1rdbms)

使用 **垂直擴展**:

* 只要單純的選擇規格比較好的機器即可
* 透過以下指標和方式來關注何時需要進行擴展
    * 使用基本的監控指標來決定系統瓶頸：CPU、記憶體、IO、Network
    * 使用 CloudWatch、top、nagios、statsd、 graphite 等工具
* 垂直擴展可能是成本很高的
* 垂直擴展不具備容錯轉移/高可用的特性

*其他的選擇或相關細節：*

* 相對於**垂直擴展**的另一個選擇是 [**水平擴展**](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E6%B0%B4%E5%B9%B3%E6%93%B4%E5%B1%95)

#### 從 SQL 資料庫開始，並且將 NoSQL 納入考量

我們的假設是這個系統是使用關連式資料，所以我們可以從一台 **MySQL 資料庫**開始規化。

*其他的選擇或相關細節：*

* 參考 [關聯式資料庫 (RDBMS)](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E9%97%9C%E9%80%A3%E5%BC%8F%E8%B3%87%E6%96%99%E5%BA%AB%E7%AE%A1%E7%90%86%E7%B3%BB%E7%B5%B1rdbms) 章節
* 討論使用 [SQL 或 NoSQL](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#sql-%E6%88%96-nosql) 的理由

#### 指定一個公開的 IP 位置

* IP 可以提供一個公開的 IP 位置，不會隨著你重開機 IP 就變更
* 綁定一個 IP 位置可以幫助你的服務具有容錯功能，當發生問題時，只要指向另外一個 IP 位置即可

#### 使用 DNS

使用像是 Route 53 這樣的 **DNS** 服務來將域名指向伺服器的公開 IP 位置。

*其他的選擇或相關細節：*

* 參考 [域名系統](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E5%9F%9F%E5%90%8D%E7%B3%BB%E7%B5%B1) 章節

#### 保護網頁伺服器

* 在防火牆部分只開啟需要的 port
    * 伺服器只允許來自底下幾個 port 得請求
        * HTTP 80
        * HTTPS 443
        * 只允許某些白名單上的 IP 位置使用 22 port 來 ssh 到這台機器
    * 避免伺服器任意的對外連線

*其他的選擇或相關細節：*

* 參考 [資訊安全](https://github.com/kevingo/system-design-primer-zh-tw/blob/master/README-zh-TW.md#%E8%B3%87%E8%A8%8A%E5%AE%89%E5%85%A8) 章節

## 步驟四：擴展你的設計

> 根據你設定的限制條件，提出目前設計架構上的瓶頸，並提出解決方法

### Users+

![Imgur](http://i.imgur.com/rrfjMXB.png)

#### 假設

我們的使用者數量開始增加，一台機器的負擔越來越大。根據我們的**監控與負載測試**顯示， **MySQL 資料庫**的記憶體和 CPU 資源負擔逐漸增加，同時使用者的資料佔滿了我們的硬碟。

我們現在已經可以使用**垂直擴展**的方式來解決這樣的問題，但是，這種方式成本很高，而且它無法針對單ㄧ**MySQL 資料庫**或**網頁伺服器**來進行擴展。

#### 目標

* 減輕單一機器的負擔，並且允許單獨縮放
    * 將靜態的內容單獨儲存在**物件儲存資料庫**中。
    * 將 **MySQL 資料庫**移到單獨的機器
* 缺點
    * 這個改變會增加系統複雜度，同時**網頁伺服器**會需要可以連接到**物件儲存資料庫**和 **MySQL 資料庫**。
    * 必須採取額外的安全措施來保護新的元件的安全
    * AWS 的成本可能也會增加，但同時你也需要考量自己管理類似系統的成本

#### Store static content separately

* Consider using a managed **Object Store** like S3 to store static content
    * Highly scalable and reliable
    * Server side encryption
* Move static content to S3
    * User files
    * JS
    * CSS
    * Images
    * Videos

#### Move the MySQL database to a separate box

* Consider using a service like RDS to manage the **MySQL Database**
    * Simple to administer, scale
    * Multiple availability zones
    * Encryption at rest

#### Secure the system

* Encrypt data in transit and at rest
* Use a Virtual Private Cloud
    * Create a public subnet for the single **Web Server** so it can send and receive traffic from the internet
    * Create a private subnet for everything else, preventing outside access
    * Only open ports from whitelisted IPs for each component
* These same patterns should be implemented for new components in the remainder of the exercise

*Trade-offs, alternatives, and additional details:*

* See the [Security](https://github.com/donnemartin/system-design-primer#security) section

### Users++

![Imgur](http://i.imgur.com/raoFTXM.png)

#### Assumptions

Our **Benchmarks/Load Tests** and **Profiling** show that our single **Web Server** bottlenecks during peak hours, resulting in slow responses and in some cases, downtime.  As the service matures, we'd also like to move towards higher availability and redundancy.

#### Goals

* The following goals attempt to address the scaling issues with the **Web Server**
    * Based on the **Benchmarks/Load Tests** and **Profiling**, you might only need to implement one or two of these techniques
* Use [**Horizontal Scaling**](https://github.com/donnemartin/system-design-primer#horizontal-scaling) to handle increasing loads and to address single points of failure
    * Add a [**Load Balancer**](https://github.com/donnemartin/system-design-primer#load-balancer) such as Amazon's ELB or HAProxy
        * ELB is highly available
        * If you are configuring your own **Load Balancer**, setting up multiple servers in [active-active](https://github.com/donnemartin/system-design-primer#active-active) or [active-passive](https://github.com/donnemartin/system-design-primer#active-passive) in multiple availability zones will improve availability
        * Terminate SSL on the **Load Balancer** to reduce computational load on backend servers and to simplify certificate administration
    * Use multiple **Web Servers** spread out over multiple availability zones
    * Use multiple **MySQL** instances in [**Master-Slave Failover**](https://github.com/donnemartin/system-design-primer#master-slave-replication) mode across multiple availability zones to improve redundancy
* Separate out the **Web Servers** from the [**Application Servers**](https://github.com/donnemartin/system-design-primer#application-layer)
    * Scale and configure both layers independently
    * **Web Servers** can run as a [**Reverse Proxy**](https://github.com/donnemartin/system-design-primer#reverse-proxy-web-server)
    * For example, you can add **Application Servers** handling **Read APIs** while others handle **Write APIs**
* Move static (and some dynamic) content to a [**Content Delivery Network (CDN)**](https://github.com/donnemartin/system-design-primer#content-delivery-network) such as CloudFront to reduce load and latency

*Trade-offs, alternatives, and additional details:*

* See the linked content above for details

### Users+++

![Imgur](http://i.imgur.com/OZCxJr0.png)

**Note:** **Internal Load Balancers** not shown to reduce clutter

#### Assumptions

Our **Benchmarks/Load Tests** and **Profiling** show that we are read-heavy (100:1 with writes) and our database is suffering from poor performance from the high read requests.

#### Goals

* The following goals attempt to address the scaling issues with the **MySQL Database**
    * Based on the **Benchmarks/Load Tests** and **Profiling**, you might only need to implement one or two of these techniques
* Move the following data to a [**Memory Cache**](https://github.com/donnemartin/system-design-primer#cache) such as Elasticache to reduce load and latency:
    * Frequently accessed content from **MySQL**
        * First, try to configure the **MySQL Database** cache to see if that is sufficient to relieve the bottleneck before implementing a **Memory Cache**
    * Session data from the **Web Servers**
        * The **Web Servers** become stateless, allowing for **Autoscaling**
    * Reading 1 MB sequentially from memory takes about 250 microseconds, while reading from SSD takes 4x and from disk takes 80x longer.<sup><a href=https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know>1</a></sup>
* Add [**MySQL Read Replicas**](https://github.com/donnemartin/system-design-primer#master-slave-replication) to reduce load on the write master
* Add more **Web Servers** and **Application Servers** to improve responsiveness

*Trade-offs, alternatives, and additional details:*

* See the linked content above for details

#### Add MySQL read replicas

* In addition to adding and scaling a **Memory Cache**, **MySQL Read Replicas** can also help relieve load on the **MySQL Write Master**
* Add logic to **Web Server** to separate out writes and reads
* Add **Load Balancers** in front of **MySQL Read Replicas** (not pictured to reduce clutter)
* Most services are read-heavy vs write-heavy

*Trade-offs, alternatives, and additional details:*

* See the [Relational database management system (RDBMS)](https://github.com/donnemartin/system-design-primer#relational-database-management-system-rdbms) section

### Users++++

![Imgur](http://i.imgur.com/3X8nmdL.png)

#### Assumptions

Our **Benchmarks/Load Tests** and **Profiling** show that our traffic spikes during regular business hours in the U.S. and drop significantly when users leave the office.  We think we can cut costs by automatically spinning up and down servers based on actual load.  We're a small shop so we'd like to automate as much of the DevOps as possible for **Autoscaling** and for the general operations.

#### Goals

* Add **Autoscaling** to provision capacity as needed
    * Keep up with traffic spikes
    * Reduce costs by powering down unused instances
* Automate DevOps
    * Chef, Puppet, Ansible, etc
* Continue monitoring metrics to address bottlenecks
    * **Host level** - Review a single EC2 instance
    * **Aggregate level** - Review load balancer stats
    * **Log analysis** - CloudWatch, CloudTrail, Loggly, Splunk, Sumo
    * **External site performance** - Pingdom or New Relic
    * **Handle notifications and incidents** - PagerDuty
    * **Error Reporting** - Sentry

#### Add autoscaling

* Consider a managed service such as AWS **Autoscaling**
    * Create one group for each **Web Server** and one for each **Application Server** type, place each group in multiple availability zones
    * Set a min and max number of instances
    * Trigger to scale up and down through CloudWatch
        * Simple time of day metric for predictable loads or
        * Metrics over a time period:
            * CPU load
            * Latency
            * Network traffic
            * Custom metric
    * Disadvantages
        * Autoscaling can introduce complexity
        * It could take some time before a system appropriately scales up to meet increased demand, or to scale down when demand drops

### Users+++++

![Imgur](http://i.imgur.com/jj3A5N8.png)

**Note:** **Autoscaling** groups not shown to reduce clutter

#### Assumptions

As the service continues to grow towards the figures outlined in the constraints, we iteratively run **Benchmarks/Load Tests** and **Profiling** to uncover and address new bottlenecks.

#### Goals

We'll continue to address scaling issues due to the problem's constraints:

* If our **MySQL Database** starts to grow too large, we might consider only storing a limited time period of data in the database, while storing the rest in a data warehouse such as Redshift
    * A data warehouse such as Redshift can comfortably handle the constraint of 1 TB of new content per month
* With 40,000 average read requests per second, read traffic for popular content can be addressed by scaling the **Memory Cache**, which is also useful for handling the unevenly distributed traffic and traffic spikes
    * The **SQL Read Replicas** might have trouble handling the cache misses, we'll probably need to employ additional SQL scaling patterns
* 400 average writes per second (with presumably significantly higher peaks) might be tough for a single **SQL Write Master-Slave**, also pointing to a need for additional scaling techniques

SQL scaling patterns include:

* [Federation](https://github.com/donnemartin/system-design-primer#federation)
* [Sharding](https://github.com/donnemartin/system-design-primer#sharding)
* [Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL Tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)

To further address the high read and write requests, we should also consider moving appropriate data to a [**NoSQL Database**](https://github.com/donnemartin/system-design-primer#nosql) such as DynamoDB.

We can further separate out our [**Application Servers**](https://github.com/donnemartin/system-design-primer#application-layer) to allow for independent scaling.  Batch processes or computations that do not need to be done in real-time can be done [**Asynchronously**](https://github.com/donnemartin/system-design-primer#asynchronism) with **Queues** and **Workers**:

* For example, in a photo service, the photo upload and the thumbnail creation can be separated:
    * **Client** uploads photo
    * **Application Server** puts a job in a **Queue** such as SQS
    * The **Worker Service** on EC2 or Lambda pulls work off the **Queue** then:
        * Creates a thumbnail
        * Updates a **Database**
        * Stores the thumbnail in the **Object Store**

*Trade-offs, alternatives, and additional details:*

* See the linked content above for details

## Additional talking points

> Additional topics to dive into, depending on the problem scope and time remaining.

### SQL scaling patterns

* [Read replicas](https://github.com/donnemartin/system-design-primer#master-slave-replication)
* [Federation](https://github.com/donnemartin/system-design-primer#federation)
* [Sharding](https://github.com/donnemartin/system-design-primer#sharding)
* [Denormalization](https://github.com/donnemartin/system-design-primer#denormalization)
* [SQL Tuning](https://github.com/donnemartin/system-design-primer#sql-tuning)

#### NoSQL

* [Key-value store](https://github.com/donnemartin/system-design-primer#key-value-store)
* [Document store](https://github.com/donnemartin/system-design-primer#document-store)
* [Wide column store](https://github.com/donnemartin/system-design-primer#wide-column-store)
* [Graph database](https://github.com/donnemartin/system-design-primer#graph-database)
* [SQL vs NoSQL](https://github.com/donnemartin/system-design-primer#sql-or-nosql)

### Caching

* Where to cache
    * [Client caching](https://github.com/donnemartin/system-design-primer#client-caching)
    * [CDN caching](https://github.com/donnemartin/system-design-primer#cdn-caching)
    * [Web server caching](https://github.com/donnemartin/system-design-primer#web-server-caching)
    * [Database caching](https://github.com/donnemartin/system-design-primer#database-caching)
    * [Application caching](https://github.com/donnemartin/system-design-primer#application-caching)
* What to cache
    * [Caching at the database query level](https://github.com/donnemartin/system-design-primer#caching-at-the-database-query-level)
    * [Caching at the object level](https://github.com/donnemartin/system-design-primer#caching-at-the-object-level)
* When to update the cache
    * [Cache-aside](https://github.com/donnemartin/system-design-primer#cache-aside)
    * [Write-through](https://github.com/donnemartin/system-design-primer#write-through)
    * [Write-behind (write-back)](https://github.com/donnemartin/system-design-primer#write-behind-write-back)
    * [Refresh ahead](https://github.com/donnemartin/system-design-primer#refresh-ahead)

### Asynchronism and microservices

* [Message queues](https://github.com/donnemartin/system-design-primer#message-queues)
* [Task queues](https://github.com/donnemartin/system-design-primer#task-queues)
* [Back pressure](https://github.com/donnemartin/system-design-primer#back-pressure)
* [Microservices](https://github.com/donnemartin/system-design-primer#microservices)

### Communications

* Discuss tradeoffs:
    * External communication with clients - [HTTP APIs following REST](https://github.com/donnemartin/system-design-primer#representational-state-transfer-rest)
    * Internal communications - [RPC](https://github.com/donnemartin/system-design-primer#remote-procedure-call-rpc)
* [Service discovery](https://github.com/donnemartin/system-design-primer#service-discovery)

### Security

Refer to the [security section](https://github.com/donnemartin/system-design-primer#security).

### Latency numbers

See [Latency numbers every programmer should know](https://github.com/donnemartin/system-design-primer#latency-numbers-every-programmer-should-know).

### Ongoing

* Continue benchmarking and monitoring your system to address bottlenecks as they come up
* Scaling is an iterative process
