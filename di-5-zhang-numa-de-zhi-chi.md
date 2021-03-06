# 第5章：NUMA支持

在第二節我們看過，在某些機器上，存取實體記憶體特殊區域的成本差異是視存取的源頭而定的。這種類型的硬體需要來自 OS 以及應用程式的特殊照顧。我們會以 NUMA 硬體的一些細節開頭，接著我們會涵蓋 Linux 系統核心為 NUMA 提供的一些支援。

## 5.1 NUMA硬件

非均勻記憶體架構正變得越來越普遍。在最簡單的 NUMA 類型中，一個處理器能夠擁有本地記憶體（見圖 2.3），存取它比存取其它處理器的本地記憶體還便宜。對這種類型的 NUMA 系統而言，成本差異並不大––即，NUMA 因子很小。

NUMA 也會––而且尤其會––用在大機器中。我們已經描述過擁有多個存取相同記憶體的處理器的問題。對商用硬體而言，所有處理器都會共享相同的北橋（此刻忽略 AMD Opteron NUMA 節點，它們有它們自己的問題）。這使得北橋成為一個嚴重的瓶頸，因為_所有_記憶體流量都會經過它。當然，大機器能夠使用客製的硬體來代替北橋，但除非使用的記憶體晶片擁有多個埠––即，它們能夠從多條匯流排使用––不然依舊有個瓶頸在。多埠 RAM 很複雜、而且建立與支援起來很昂貴，因此幾乎不會被使用。

下一個複雜度上的改進為 AMD 使用的模型，其中的互連機制（在 AMD 情況下為超傳輸〔Hyper Transport〕，是它們由 Digital 授權而來的技術）為沒有直接連接到 RAM 的處理器提供了存取。能夠以這種方式組成的結構大小是有限的，除非你想要任意地增加直徑（diameter）（即，任意兩節點之間的最大距離）。

[![&#x5716; 5.1&#xFF1A;&#x8D85;&#x7ACB;&#x65B9;&#x9AD4;](https://github.com/jason2506/cpumemory.zh-tw/raw/master/assets/figure-5.1.png)](https://github.com/jason2506/cpumemory.zh-tw/blob/master/assets/figure-5.1.png)圖 5.1：超立方體

一種連接節點的有效拓撲（topology）為超立方體（hypercube），其將節點的數量限制在 $$ 2^{C} $$，其中 $$ C $$ 為每個節點擁有的交互連接介面的數量。以所有有著 $$ 2^{n} $$ 個 CPU 與 $$ n $$ 條交互連接的系統而言，超立方體擁有最小的直徑。圖 5.1 顯示了前三種超立方體。每個超立方體擁有絕對最小（the absolute minimum）的直徑 $$ C $$。AMD 第一世代的 Opteron 處理器，每個處理器擁有三條超傳輸連結。至少有一個處理器必須有個附屬在一條連結上的南橋，代表––目前而言––一個 $$ C = 2 $$ 的超立方體能夠直接且有效率地實作。下個世代將在某個時間點擁有四條連結，屆時將可能有 $$ C = 3 $$ 的超立方體。

不過，這不代表無法支援更大的處理器集合體（accumulation）。有些公司已經開發出能夠使用更大的處理器集合的 crossbar（例如，Newisys 的 Horus）。但這些 crossbar 提高了 NUMA 因子，而且在一定數量的處理器上便不再有效。

下一個改進為連接 CPU 的群組，並為它們全體實作一個共享的記憶體。所有這類系統都需要特製化的硬體，絕不是商用系統。 Such designs exist at several levels of complexity. 一個仍然十分接近於商用機器的系統為 IBM x445 以及類似的機器。它們能夠當作有著 x86 與 x86-64 的普通 4U、8 路機器購買。兩台（某些時候高達四台）這種機器就能夠被連接起來運作，如同一台有著共享記憶體的機器。使用的交互連接引入了一個 OS––以及應用程式––必須納入考量的重要的 NUMA 因子。

在光譜的另一端，像 SGI 的 Altix 這樣的機器是專門被設計來互連的。SGI 的 NUMAlink 互連結構非常地快，同時擁有很短的等待時間；兩個特性對於高效能計算（high-performance computing，HPC）都是必要條件，尤其是在使用訊息傳遞介面（Message Passing Interface，MPI）的時候。當然，缺點是，這種精密與特製化是非常昂貴的。它們令合理地低的 NUMA 因子成為可能，但以這些機器能擁有的 CPU 數量（幾千個）以及有限的互連能力，NUMA 因子實際上是動態的，而且可能因工作負載而達到不可接受的程度。

更常使用的解決方法是，使用高速網路將許多商用機器連接起來，組成一個集群（cluster）。不過，這些並非 NUMA 機器；它們沒有實作共享的位址空間，因此不會歸於這裡討論的任何一個範疇中。

## 5.2 OS对NUMA的支持

為了支援 NUMA 機器，OS 必須將記憶體分散式的性質納入考量。舉例來說，若是一個行程執行在一個給定的處理器上，被指派給行程位址空間的實體 RAM 理想上應該要來自本地記憶體。否則每條指令都必須為了程式碼與資料去存取遠端的記憶體。有些僅存於 NUMA 機器的特殊情況要被考慮進去。DSO 的文字區段（text segment）在一台機器的實體 RAM 中通常正好出現一次。但若是 DSO 被所有 CPU 上的行程與執行緒用到（例如，像 `libc` 這類基本的執行期函式庫），這表示並非一些、而是全部的處理器都必須擁有遠端的位址。OS 理想上會將這種 DSO「映像（mirror）」到每個處理器的實體 RAM 中，並使用本地的副本。這並非一種最佳化，而是個必要條件，而且通常難以實作。它可能沒有被支援、或者僅以有限的形式支援。

為了避免情況變得更糟，OS 不該將一個行程或執行緒從一個節點遷移到另一個。OS 應該已經試著避免在一般的多處理器機器上遷移行程，因為從一個處理器遷移到另一個處理器意味著快取內容遺失了。若是負載分配需要將一個行程或執行緒遷出一個處理器，OS 通常能夠挑選任一個擁有足夠剩餘容量的新處理器。在 NUMA 環境中，新處理器的選擇受到了稍微多一些的限制。對於行程正在使用的記憶體，新選擇的處理器不該有比舊的處理器還高的存取成本；這限制了目標的清單。若是沒有符合可用標準的空閒處理器，OS 除了遷移到記憶體存取更為昂貴的處理器以外別無他選。

在這種情況下，有兩種可能的方法。首先，可以期盼這種情況是暫時的，而且行程能夠被遷回到一個更合適的處理器上。或者，OS 也能夠將行程的記憶體遷移到更靠近新使用的處理器的實體分頁上。這是個相當昂貴的操作。可能得要複製大量的記憶體，儘管不必在一個步驟中完成。當發生這種情況的時候，行程必須––至少短暫地––中止，以正確地遷移對舊分頁的修改。有了讓分頁遷移有效又快速，有整整一串其它的必要條件。簡而言之，OS 應該避免它，除非它是真的有必要的。

一般來說，不能夠假設在一台 NUMA 機器上的所有行程都使用等量的記憶體，使得––由於遍及各個處理器的行程的分佈––記憶體的使用也會被均等地分配。事實上，除非執行在機器上的應用程式非常特殊（在 HPC 世界中很常見，但除此之外則否），不然記憶體的使用是非常不均等的。某些應用程式會使用巨量的記憶體，其餘的幾乎不用。若總是分配產生請求的處理器本地的記憶體，這將會––或早或晚––造成問題。系統最終將會耗盡執行大行程的節點本地的記憶體。

為了應對這些嚴重的問題，記憶體––預設情況下––不會只分配給本地的節點。為了利用所有系統的記憶體，預設策略是條帶化（stripe）記憶體。這保證了所有系統記憶體的同等使用。作為一個副作用，有可能變得能在處理器之間自由遷移行程，因為––平均而言––對於所有用到的記憶體的存取成本沒有改變。由於很小的 NUMA 因子，條帶化是可以接受的，但仍不是最好的（見 5.4 節的數據）。

這是個幫助系統避免嚴重問題、並在普通操作下更為能夠預測的劣化（pessimization）。但它降低了整體的系統效能，在某些情況下尤為顯著。這即是 Linux 允許每個行程選擇記憶體分配規則的原因。一個行程能夠為它自己以及它的子行程選擇不同的策略。我們將會在第六節介紹能用於此的介面。

## 5.3 被發布的資訊

系統核心透過 `sys` 虛擬檔案系統（sysfs）將處理器快取的資訊發布在

`/sys/devices/system/cpu/cpu*/cache`

在 6.2.1 節，我們會看到能用來查詢不同快取大小的介面。這裡重要的是快取的拓樸。上面的目錄包含了列出 CPU 擁有的不同快取資訊的子目錄（叫做 `index*`）。檔案 `type`、`level`、與 `shared_cpu_map` 是在這些目錄中與拓樸有關的重要檔案。一個 Intel Core 2 QX6700 的資訊看起來就如表 5.1。

|  | `type` | `level` | `shared_cpu_map` |  |
| :--- | :--- | :--- | :--- | :--- |
| `cpu0` | `index0` | Data | 1 | 00000001 |
| `index1` | Instruction | 1 | 00000001 |  |
| `index2` | Unified | 2 | 00000003 |  |
| `cpu1` | `index0` | Data | 1 | 00000002 |
| `index1` | Instruction | 1 | 00000002 |  |
| `index2` | Unified | 2 | 00000003 |  |
| `cpu2` | `index0` | Data | 1 | 00000004 |
| `index1` | Instruction | 1 | 00000004 |  |
| `index2` | Unified | 2 | 0000000c |  |
| `cpu3` | `index0` | Data | 1 | 00000008 |
| `index1` | Instruction | 1 | 00000008 |  |
| `index2` | Unified | 2 | 0000000c |  |

表 5.1：Core 2 CPU 快取的 `sysfs` 資訊

這份資料的意義如下：

* 每顆核心\[^25\]擁有三個快取：L1i、L1d、L2。
* L1d 與 L1i 快取沒有被任何其它的核心所共享––每顆核心有它自己的一組快取。這是由 `shared_cpu_map` 中的位元圖（bitmap）只有一個被設置的位元所暗示的。
* `cpu0` 與 `cpu1` 的 L2 快取是共享的，正如 `cpu2` 與 `cpu3` 上的 L2 一樣。

若是 CPU 有更多快取階層，也會有更多的 `index*` 目錄。

|  | `type` | `level` | `shared_cpu_map` |  |
| :--- | :--- | :--- | :--- | :--- |
| `cpu0` | `index0` | Data | 1 | 00000001 |
| `index1` | Instruction | 1 | 00000001 |  |
| `index2` | Unified | 2 | 00000001 |  |
| `cpu1` | `index0` | Data | 1 | 00000002 |
| `index1` | Instruction | 1 | 00000002 |  |
| `index2` | Unified | 2 | 00000002 |  |
| `cpu2` | `index0` | Data | 1 | 00000004 |
| `index1` | Instruction | 1 | 00000004 |  |
| `index2` | Unified | 2 | 00000004 |  |
| `cpu3` | `index0` | Data | 1 | 00000008 |
| `index1` | Instruction | 1 | 00000008 |  |
| `index2` | Unified | 2 | 00000008 |  |
| `cpu4` | `index0` | Data | 1 | 00000010 |
| `index1` | Instruction | 1 | 00000010 |  |
| `index2` | Unified | 2 | 00000010 |  |
| `cpu5` | `index0` | Data | 1 | 00000020 |
| `index1` | Instruction | 1 | 00000020 |  |
| `index2` | Unified | 2 | 00000020 |  |
| `cpu6` | `index0` | Data | 1 | 00000040 |
| `index1` | Instruction | 1 | 00000040 |  |
| `index2` | Unified | 2 | 00000040 |  |
| `cpu7` | `index0` | Data | 1 | 00000080 |
| `index1` | Instruction | 1 | 00000080 |  |
| `index2` | Unified | 2 | 00000080 |  |

表 5.2：Opteron CPU 快取的 `sysfs` 資訊

對於一個四槽、雙核的 Opteron 機器，快取資訊看起來如表 5.2。可以看出這些處理器也有三種快取：L1i、L1d、L2。沒有核心共享任何階層的快取。這個系統有趣的部分在於處理器拓樸。少了這個額外資訊，就無法理解快取資料。`sys` 檔案系統將這個資訊擺在下面這個檔案

`/sys/devices/system/cpu/cpu*/topology`

表 5.3 顯示了在 SMP Opteron 機器的這個階層裡頭的令人感興趣的檔案。

|  | `physical_package_id` | `core_id` | `core_siblings` | `thread_siblings` |
| :--- | :--- | :--- | :--- | :--- |
| `cpu0` | 0 | 0 | 00000003 | 00000001 |
| `cpu1` | 1 | 00000003 | 00000002 |  |
| `cpu2` | 1 | 0 | 0000000c | 00000004 |
| `cpu3` | 1 | 0000000c | 00000008 |  |
| `cpu4` | 2 | 0 | 00000030 | 00000010 |
| `cpu5` | 1 | 00000030 | 00000020 |  |
| `cpu6` | 3 | 0 | 000000c0 | 00000040 |
| `cpu7` | 1 | 000000c0 | 00000080 |  |

表 5.3：Opteron CPU 拓樸的 `sysfs` 資訊

將表 5.2 與 5.3 擺在一起，我們能夠發現

* 沒有 CPU 擁有超執行緒（`thethread_siblings` 位元圖有一個位元被設置）、
* 這個系統實際上共有四個處理器（`physical_package_id` 0 到 3）、
* 每個處理器有兩顆核心、以及
* 沒有核心共享任何快取。

這正好與較早期的 Opteron 一致。

目前為止提供的資料中完全缺少的是，有關這台機器上的 NUMA 性質的資訊。任何 SMP Opteron 機器都是一台 NUMA 機器。為了這份資料，我們必須看看在 NUMA 機器上存在的 `sys` 檔案系統的另一個部分，即下面的階層中

`/sys/devices/system/node`

這個目錄包含在系統上的每個 NUMA 節點的子目錄。在特定節點的目錄中有許多檔案。在前兩張表中描述的 Opteron 機器的重要檔案與它們的內容顯示在表 5.4。

|  | `cpumap` | `distance` |
| :--- | :--- | :--- |
| `node0` | 00000003 | 10 20 20 20 |
| `node0` | 0000000c | 20 10 20 20 |
| `node2` | 00000030 | 20 20 10 20 |
| `node3` | 000000c0 | 20 20 20 10 |

表 5.4：Opteron 節點的 `sysfs` 資訊

這個資訊將所有的一切連繫在一起；現在我們有個機器架構的完整輪廓了。我們已經知道機器擁有四個處理器。每個處理器構成它自己的節點，能夠從 `node*` 目錄的 `cpumap` 檔案中的值裡頭設置的位元看出來。在那些目錄的 `distance` 檔案包含一組值，一個值代表一個節點，表示在各個節點上存取記憶體的成本。在這個例子中，所有本地記憶體存取的成本為 10，所有對任何其它節點的遠端存取的成本為 20。\[^26\]這表示，即使處理器被組織成一個二維超立方體（見圖 5.1），在沒有直接連接的處理器之間存取也沒有比較貴。成本的相對值應該能用來作為存取時間的實際差距的估計。所有這些資訊的準確性是另一個問題了。

\[^25\]: `cpu0` 到 `cpu3` 為核心的知識來自於另一個將會簡短介紹的地方。

\[^26\]: 順帶一提，這是不正確的。這個 ACPI 資訊明顯是錯的，因為––雖然用到的處理器擁有三條連貫的超傳輸連結––至少一個處理器必須被連接到一個南橋上。至少一對節點必須因此有比較大的距離。

## 5.4 遠端存取成本

[![&#x5716; 5.2&#xFF1A;&#x591A;&#x7BC0;&#x9EDE;&#x7684;&#x8B80;&#xFF0F;&#x5BEB;&#x6548;&#x80FD;](https://github.com/jason2506/cpumemory.zh-tw/raw/master/assets/figure-5.2.png)](https://github.com/jason2506/cpumemory.zh-tw/blob/master/assets/figure-5.2.png)圖 5.2：多節點的讀／寫效能

不過，距離是有關係的。AMD 在 \[1\] 提供了一台四槽機器的 NUMA 成本的文件。寫入操作的數據顯示在圖 5.2。寫入比讀取還慢，這並不讓人意外。有趣的部分在於 1 與 2 跳（1- and 2-hop）情況下的成本。兩個 1 跳的成本實際上有略微不同。細節見 \[1\]。2 跳讀取與寫入（分別）比 0 跳讀取慢了 30% 與 49%。2 跳寫入比 0 跳寫入慢了 32%、比 1 跳寫入慢了 17%。處理器與記憶體節點的相對位置能夠造成很大的差距。來自 AMD 下個世代的處理器將會以每個處理器四條連貫的超傳輸連結為特色。在這個例子中，一台四槽機器的直徑會是一。但有八個插槽的話，同樣的問題又––來勢洶洶地––回來了，因為一個有著八個節點的超立方體的直徑為三。

所有這些資訊都能夠取得，但用起來很麻煩。在 6.5 節，我們會看到較容易存取與使用這個資訊的介面。

```text
00400000 default file=/bin/cat mapped=3 N3=3
00504000 default file=/bin/cat anon=1 dirty=1 mapped=2 N3=2
00506000 default heap anon=3 dirty=3 active=0 N3=3
38a9000000 default file=/lib64/ld-2.4.so mapped=22 mapmax=47 N1=22
38a9119000 default file=/lib64/ld-2.4.so anon=1 dirty=1 N3=1
38a911a000 default file=/lib64/ld-2.4.so anon=1 dirty=1 N3=1
38a9200000 default file=/lib64/libc-2.4.so mapped=53 mapmax=52 N1=51 N2=2
38a933f000 default file=/lib64/libc-2.4.so
38a943f000 default file=/lib64/libc-2.4.so anon=1 dirty=1 mapped=3 mapmax=32 N1=2 N3=1
38a9443000 default file=/lib64/libc-2.4.so anon=1 dirty=1 N3=1
38a9444000 default anon=4 dirty=4 active=0 N3=4
2b2bbcdce000 default anon=1 dirty=1 N3=1
2b2bbcde4000 default anon=2 dirty=2 N3=2
2b2bbcde6000 default file=/usr/lib/locale/locale-archive mapped=11 mapmax=8 N0=11
7fffedcc7000 default stack anon=2 dirty=2 N3=2
```

圖 5.3：`/proc/`**`PID`**`/numa_maps` 的內容

系統提供的最後一塊資訊就在行程自身的狀態中。能夠確定記憶體映射檔、寫時複製（Copy-On-Write，COW）\[^27\]分頁與匿名記憶體（anonymous memory）是如何散布在系統中的節點上的。系統核心為每個處理器提供一個虛擬檔（pseudo-file） `/proc/`**`PID`**`/numa_maps`，其中 **PID** 為行程的 ID ，如圖 5.3 所示。檔案中的重要資訊為 `N0` 到 `N3` 的值，它們表示為節點 0 到 3 上的記憶體區域分配的分頁數量。一個可靠的猜測是，程式是執行在節點 3 的一顆核心上。程式本身與被弄髒的分頁被分配在這個節點上。唯讀映射，像是 `ld-2.4.so` 與 `libc-2.4.so` 的第一次映射、以及共享檔案 `locale-archive` 是被分配在其它節點上的。

如同我們已經在圖 5.2 看到的，當橫跨節點操作時，1 與 2 跳讀取的效能分別掉了 9% 與 30%。對執行來說，這種讀取是必須的，而且若是沒有命中 L2 快取的話，每個快取行都會招致這些額外成本。若是記憶體離處理器很遠，對超過快取大小的大工作負載而言，所有量測的成本都必須提高 9%／30%。

[![&#x5716; 5.4&#xFF1A;&#x5728;&#x9060;&#x7AEF;&#x8A18;&#x61B6;&#x9AD4;&#x64CD;&#x4F5C;](https://github.com/jason2506/cpumemory.zh-tw/raw/master/assets/figure-5.4.png)](https://github.com/jason2506/cpumemory.zh-tw/blob/master/assets/figure-5.4.png)圖 5.4：在遠端記憶體操作

為了看到在現實世界的影響，我們能夠像 3.5.1 節一樣測量頻寬，但這次使用的是在遠端、相距一跳的節點上的記憶體。這個測試相比於使用本地記憶體的數據的結果能在圖 5.4 中看到。數字在兩個方向都有一些大起伏，這是一個測量多執行緒程式的問題所致，能夠忽略。在這張圖上的重要資訊是，讀取操作總是慢了 20%。這明顯慢於圖 5.2 中的 9%，這極有可能不是連續讀／寫操作的數字，而且可能與較舊的處理器修訂版本有關。只有 AMD 知道了。

以塞得進快取的工作集大小而言，寫入與複製操作的效能也慢了 20%。當工作集大小超過快取大小時，寫入效能不再顯著地慢於本地節點上的操作。互連的速度足以跟上記憶體的速度。主要因素是花費在等待主記憶體的時間。

\[^27\]: 當一個記憶體分頁起初有個使用者，然後必須被複製以允許獨立的使用者時，寫時複製是一種經常在 OS 實作用到的方法。在許多情境中，複製––起初或完全––是不必要的。在這種情況下，只在任何一個使用者修改記憶體的時候複製是合理的。作業系統攔截寫入操作、複製記憶體分頁、然後允許寫入指令繼續執行。





