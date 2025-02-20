### 前言

鐵人賽的最後，分享一下我在今年考過 CKA 的攻略：

* CKA 簡介

* 報名考試 & 考試預約

* 準備考試與學習資源

* 怎麼知道自己準備好了？

* 考試技巧與注意事項

* 考試當天的流程

* 考題 Brain Dump 與心得

* 附錄：CKA 考試時常用的官方文件清單

## CKA 簡介

`CKA` 是 Certified Kubernetes Administrator 的縮寫，是由 *Cloud Native Computing Foundation* (CNCF) 所提供的 Kubernetes 認證考試，主要針對 Kubernetes 的操作與管理能力進行考核。

* **考試時間**：2 小時
* **考試方式**：線上考試、線上考官監考
* **題目數量**：15-20 實作題 (我這次是 17 題)
* **通過分數**：66 分，滿分 100 分。
* **重考次數**：1 次 (第一次沒過能免費再考一次)
* **結果通知**：考試結束後 24 小時內
* **Open Book**：可以查閱[官方文件](https://kubernetes.io/docs/)。(不能查看其他網站)


> [CKA 官網](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)

![https://ithelp.ithome.com.tw/upload/images/20240919/20168692C7muecLVZV.png](https://ithelp.ithome.com.tw/upload/images/20240919/20168692C7muecLVZV.png)

CKA 的考試範圍與比重如下：

Domain | weight
-------|--------
Storage | 10%
Troubleshooting | 30%
Workloads & Scheduling | 15%
Cluster Architecture, Installation & Configuration | 25%
Services & Networking | 20%

上面的五大領域之下還有其他的子領域，可以參考 [CKA 官網](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/#)。(基本上這次[鐵人賽系列](https://ithelp.ithome.com.tw/users/20168692/ironman/7376)已經涵蓋了全部的考試範圍，可以參考 [Day 01](https://ithelp.ithome.com.tw/articles/10344706) 查看每個 Domain 與相關文章介紹的對應表)

## 報名考試 & 考試預約

CKA 的考試費用為 395 美元，建議等到有折扣再報名會省很多，例如我是在 2023 的的黑色星期五報名的，費用是 197.5 美金，省了一半。

> 也不一定只有黑五有折扣，可以留意一下 [Linux Foundation 的官方臉書](https://www.facebook.com/TheLinuxFoundation)，只要有優惠基本上都會發文，例如今年八月份就有「夏季優惠」，全證照 40% off。

前往 [CKA 官網](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/) 即可報名考試，報名成功後會寄 email 通知：

![https://ithelp.ithome.com.tw/upload/images/20240920/20168692A1q7kzGr5r.png](https://ithelp.ithome.com.tw/upload/images/20240920/20168692A1q7kzGr5r.png)


上面的 email 中最重要的是有一個「**My Portal**」的連結，這個連結超級重要，驗證姓名、預約考試、系統需求測試、考試須知、考試入口以及考試結果通通在這個 portal 上，建議用書籤先存起來。

另外 My Portal 的右上角會有一個 [killer.sh](https://killer.sh/) 的模擬考連結，總共提供兩次模擬考，兩次模擬考的題目皆相同，一次模擬考有 36 小時，因此總共有 72 小時的刷題時間。

> 因為只有寶貴的 72 小時，建議等到考試前幾天、準備充分了再去刷模擬考。

### 考試預約

報名成功後可以在一年內預約考試時間。在預約考試前，先在「My Protal」上確認以下幾點：

* `Verify Name`：確認這裡填寫的姓名與護照上的「英文姓名」**完全一致**。(考試會用護照來驗證身份，名字寫錯就不用考了)

* `Check System Requirements`：確認用來考試的電腦是否符合要求。

確認完後，在「My Portal」上，點選「`Schedule an Exam`」就可以開始預約了。開頭可以選擇「考試語言」，雖然有中文能選，但還是建議選英文，因為怕中文翻譯有時候怪怪的，英文考題能最清楚的表達題意、用字其實也不難。

在時間安排上，能預約的考試時段相當多，通常都能找到合適自己的考試時間。預約完成後，會寄 email 通知，裡面會說明只要不是考試開始前 24 小時，都可以隨時「取消」或「更改考試時間」：

![https://ithelp.ithome.com.tw/upload/images/20240919/201686927XS5y1Pmv6.png](https://ithelp.ithome.com.tw/upload/images/20240919/201686927XS5y1Pmv6.png)

> 在預約的過程中，會提示你可以先上傳護照的照片，建議照做，因為可以節省考試當天身分驗證的時間，也可以避免鏡頭對焦不到護照、導致考官無法驗證身份的情況。

## 準備考試與學習資源

報名完成後就可以開始準備考試了。準備考試有以下三大步驟：

#### **Step 1：學習考試內容**

先看看是否已經掌握了考試範圍內的知識點，如果想系統性的從頭學起，可以參考：

* *我的鐵人賽系列*：[入門 Kubernetes 到考取 CKA 證照](https://ithelp.ithome.com.tw/users/20168692/ironman/7376)

* *線上課程*：唯一推薦 Udemy 上的「[Certified Kubernetes Administrator (CKA) with Practice Tests](https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/?couponCode=ST11MT91624A)」，雖然全英文授課，不過語速偏慢、用詞簡單、口音不重，並且最重要的提供一堆的 Lab 練習，還可以反覆刷題。

> 如果你的 Udemy 帳號已經買過其他課程了，建議再辦一支新的帳號再買這個CKA課程，印象中 Udemy 新帳號的第一堂課都 399 台幣，原價買這個課程要兩千多。


#### **Step 2：刷題**

學習基礎知識後，就可以開始刷題了。正式考試時是可以查[官網](https://kubernetes.io/docs/home/)的，在刷題時不會的操作或 **yaml 範例**盡量在官網上找答案，「熟悉官網文件、快速找到答案」是刷題一定要培養的能力。

> 查官網時可以善用「Ctrl + f」來搜尋關鍵字。

以下是我自己刷過的免費題庫：

* [killercoda 的 CKA 專區](https://killercoda.com/cka)：總共有 120 幾題的實作，難度我覺得和考試差不多，蠻推薦的。

* [CKA Exercises](https://github.com/chadmcrowell/CKA-Exercises/blob/main/cluster-architecture.md)：CKA 小練習，不太算題目，但都是考試時常用的操作。

付費的題目我只刷過 [Udemy 課程]((https://www.udemy.com/course/certified-kubernetes-administrator-with-practice-tests/?couponCode=ST11MT91624A))附贈的 Lab，如果你覺得免費的題庫刷完了還是不安心可以買這個，直接跳過它的課程開始做 Lab。

> 授課單元搭配的題目通常都很簡單，主要是讓你熟悉操作，不過最後面還有三個模擬試題還是蠻有參考價值的，尤其是一些關於 jsonpath 的操作。

刷題的過程中難免會出錯，我當時會針對錯誤的題目回去複習相關知識點，然後把這些知識點整理在一起，在模擬考或正式考試之前複習。另外我也會複習特定操作在官方文件中的位置，這裡是我整理的清單：「[CKA 常用的官方文件](https://github.com/michaelchen1225/CKA-note/blob/main/%E9%99%84%E9%8C%84/CKA%20%E5%B8%B8%E7%94%A8%E7%9A%84%E5%AE%98%E6%96%B9%E6%96%87%E4%BB%B6.md)」。

#### **Step 3：模擬考**

刷完題目、補齊知識點、整理錯誤後，在考試前一周就可以來寫 killer.sh 的模擬考，從「My Protal」右上角即可進入。

這份模擬考的操作環境和正式考試時很相似，可以藉機適應考試環境。在難度方面，考試題目難度還高，但原因倒不是因為完全不會解，而是操作的複雜度較高、且要在兩個小時內寫完 25 題，相當考驗官方文件的熟練度。

> 在模擬考時，把它當成正式的考試去考，盡量不要查答案或筆記。

如果第一次考出來成績不理想很正常(我第一次考完才 68/125)，把錯誤的知識點學起來就好。

在複習、紀錄錯誤的知識點之外，模擬考考完後會有一份官方解答，大力推薦好好閱讀，其中的解題的流程相當清楚且實用，即使是做對的題目也不要跳過，解答中可能有不知道的操作技巧或解題思路。

## 怎麼知道自己準備好了？

衡量指標相當簡單：

> 做完以上的三大步驟後，如果你能在兩個小時內、僅參考官方文件的情況下，killer.sh 模擬考拿到 110 分上下就穩了。

以我的例子來說：

* 本來安排 8/17 考試，不過在八月初考模擬考時，第一次 68、第二次 110、第三次滿分，所以果斷將考試提前到 8/4，最後正式考試時也是拿到滿分。

此外，不用擔心會考「安裝」 CNI、Ingress controller、metrics server 等等需要背「網址」的操作，需要的話題目會先安裝好，你只需要懂如何操作即可。

*以 metrics server 為例，考試時只需知道如何使用 `kubectl top` 即可，環境會先安裝好 metrics server*。

## 考試技巧與注意事項

知道自己準備好後，就可以準備來參加考試了。以下是一些考試的小技巧：

* kubectl 的 `alias=k` 和 `bash completion` 預設已經設好了，多多利用可以節省考試時間。

* 忘記指令用法就多用 `-h`，通常會有範例可以參考。例如忘記怎麼建立 ingress 也不一定要馬上查官網，-h 就說得蠻清楚的：

```bash
kubectl create ingress -h
```
```shell
...(略)
Examples:
  # Create a single ingress called 'simple' that directs requests to foo.com/bar to svc
  # svc1:8080 with a TLS secret "my-cert"
  kubectl create ingress simple --rule="foo.com/bar=svc1:8080,tls=my-cert"
  
  # Create a catch all ingress of "/path" pointing to service svc:port and Ingress Class as
"otheringress"
  kubectl create ingress catch-all --class=otheringress --rule="/path=svc:port"
  
  # Create an ingress with two annotations: ingress.annotation1 and ingress.annotations2
  kubectl create ingress annotated --class=default --rule="foo.com/bar=svc:port" \
  --annotation ingress.annotation1=foo \
  --annotation ingress.annotation2=bla
 ...(略)
 ```

* 產生 yaml 範例：

```bash
export do="--dry-run=client -o yaml"
kubectl <command> $do > example.yaml
```

* DaemonSet 的 yaml 不用特別查，用 Deployment 的 yaml 改一改就好。(可參考 [Day 17](https://ithelp.ithome.com.tw/articles/10347876))

* 考試時會有好幾組 cluster，寫題目時記得要切換到正確的 cluster：(通常題目敘述中會有底下的指令讓你 copy)

```bash
kubectl config use-context <cluster-name>
```

* 用題號命名該題的 yaml 檔，例如第六題的題目就命名為 `06.yaml`，這樣在檢查時會比較方便。

* 文字終端中的複製與貼上：`Ctrl + Shift + C` 與 `Ctrl + Shift + V`。

* 遇到以下題目建議先標記然後跳過，寫完有空再說，畢竟及格分數只要 66：
  * 題目敘述看完後完全沒有頭緒
  * 做 10 分鐘以上做不出來的題目

* 考試介面有提供「Flag」功能，可以用來標記特定的題目。(但是斷線重連後會消失)

* 考試介面有提供 NOTE 功能，可以用來標記不會的題目、筆記等等。(不用擔心斷線重連後會消失)

* 題目會提供該題相關的官方文件連結，雖然不一定有用，但直接點就可以到官網了。

萬事俱備後，考試的注意事項如下：

**Check-in 前**

  * 至少要完整的讀過一遍考試官方提供的兩份文件：
    * 「[Candidate Handbook](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2)」

    * 「[Important Instructions: CKA and CKAD](https://docs.linuxfoundation.org/tc-docs/certification/tips-cka-and-ckad)」

  * 推薦先做完「[PSI Remote Testing Tutorial](https://docs.linuxfoundation.org/tc-docs/certification/lf-handbook2/psi-remote-testing-tutorial-test-the-secure-browser-+-add-appointment-to-your-online-calendar)」，提前熟悉線上考試的流程與介面。

  * 提前準備一個房間做為考試環境，需求如下：
    * 房間必須安靜、隱密、照明清楚
    * 房間環境不能太雜亂，考生的背後最好是空白的牆壁
    * 考生不能背對光源，例如窗戶、燈光
    * 房間內不能出現除考生外的其他人
    * 桌面不能有出現電子設備、筆記

  * 可提前 30 分鐘進入「My Portal」，點選 `Take Exam` 開始 check-in 流程。

  * 備妥護照、Webcam、麥克風、電腦、網路環境。

  * 去上廁所。

**Check-in 時**

  * Check-in 時就會進入監考方提供的 browser，由一個功能是「一鍵關閉背景程式」，可以幫你關閉所有不合規定的程式。

  * 考試介面右下角會有訊息欄，用來和考官溝通。(考官不會露臉，也不會有聲音，只有文字)

  * 考官會用文字給出指示，一定要記得回應，不知道回什麼至少回個「OK」。

  * 依照考官指示，開啟螢幕同步來分享畫面。

  * 依照考官指示，開啟 WebCam 讓考官檢查環境。

  * 考官會檢查考生的臉、耳後、手腕。

  * 可以放飲用水在桌面上，裝水的容器要是透明的，必須先給考官看過。

**考試時**
   
  * 作答之前，記得檢查題目要求要在「哪個 cluster」操作。

  * 不能唸出題目、托著下巴、自言自語。

  * 臉不要離開 WebCam 範圍。

  * 可以上廁所，要跟考官申請 timeout、報備。

  * 如果斷線了不用緊張，可以重新連線。

  * 檢查完可直接交卷。

## 考試當天的流程

考試當天，我預約在 8/4 的 14:30 進行考試，這裡分享一下當天的流程：

* 14:00，提前 30 分鐘進入「My Portal」，點選 `Take Exam`，等待考官開始 check-in 流程。

* 由於我提前上傳了護照，身分驗證很快就過了，然後考官用文字傳送了一串應注意事項，我都回答「OK」。之後就是檢查環境。

* 我是用筆電考試的，因此我得舉著筆電繞來繞去，攝影的角度要喬一段時間。考試環境是我的房間，我提前將雜物搬到門外、書櫃用布料完全罩住，其他就只剩床、櫃子了，因此環境檢查也很快就過了。然後是檢查臉、耳後、手腕、桌子底下。

* 然後考官要求把手機拿出來對準攝影機，然後他請我把手機放置在考試時拿不到的地方，我離開座位，把手機放在房間最遠的櫃子上。

* 回來後因為我沒有用筆電的攝影機顯示我到底把手機放哪了，所以考官要求我給他看一下。(又拿著筆電走來走去)

* 都檢查完後，考官介紹了一下考試介面，例如筆記功能、timeouts 等等，然後問我準備好了嗎，我回答「Yes」，考官就提早讓我考試了。(預約 14:30，這時大約 14:20)

* 拿到考試環境時，我最不適應的是視窗與文字的比例，花了快五分鐘調整。

* 開始作答時，又發現從終端機與瀏覽器的切換被上方邊框擋住了，因此又花了一點時間調整。

* 調整完後就開始作答。題目都蠻簡單的，寫得很順利，全部寫完時還剩下一個小時可以檢查。

* 檢查兩遍後還剩 30 幾分鐘，我就提早交卷了。(此時大約 15:50)

考試的隔天，也就是 8/5 號的 14:50 我收到了結果通知的 email：

![https://ithelp.ithome.com.tw/upload/images/20240919/20168692iTEP3Stk9m.png](https://ithelp.ithome.com.tw/upload/images/20240919/20168692iTEP3Stk9m.png)

我就按照提示去到「My Portal」查看成績：

![https://ithelp.ithome.com.tw/upload/images/20240919/20168692XcVB3Myby4.png](https://ithelp.ithome.com.tw/upload/images/20240919/20168692XcVB3Myby4.png)

然後點「View Certificate」就可以下載證書了(證書是 PDF 檔)：

![https://ithelp.ithome.com.tw/upload/images/20240919/20168692Q6YsRatHPx.png](https://ithelp.ithome.com.tw/upload/images/20240919/20168692Q6YsRatHPx.png)

另外在 email 中也有提到我獲得了一個在 [Credly](https://info.credly.com/) 上的電子徽章：

![https://ithelp.ithome.com.tw/upload/images/20240919/20168692HgUuywL1IU.png](https://ithelp.ithome.com.tw/upload/images/20240919/20168692HgUuywL1IU.png)

[CKA: Certified Kubernetes Administrator](https://www.credly.com/badges/8bbbabf9-6f94-45e0-85db-1ea8e9d105e9/public_url)

## 考題 Brain Dump 與心得

這次的 CKA 考試總共有 17 題，這裡盡量回憶了 17 題的考點：

> 非實際考題順序

1. 建立 hostpath PV，然後建立 PVC 掛到 Pod 裡面。

2. 建立 PVC 並且掛到 Pod。

3. 某個 Worker Node 狀態是 NotReady，原因是 kubelet 沒有啟動，重啟即可。

4. 升級 cluster ，只需升級 Master Node。

5. 備份與還原 etcd。 

6. RBAC：建立 clusterrole，bind 到一個 Service Accout。

7. Scale up 一個 Deployment 到 4 個 replicas。

8. 用 label selector 搭配 `kubectl top` 找出使用最多 cpu 的 Pod。

9. 建立內含兩個容器的 Pod。

10. 建立一個函有 Sidecar 容器的 Pod，使用 emptyDir 在兩個容器之間共享資料。

11. Drain 一個 Node。

12. 找出某個 cluster 目前 ready 且可以 schedule 的 Node數量，將數量寫入一個檔案。

13. 建立 Network Policy。(不複雜，蠻基本的)

14. 建立 Ingress。(也不複雜)

15. 看某個 Pod 的 log，將 error 的訊息紀錄到一個檔案。

16. 在 Deployment 中開放 80 port 並設定 port name，然後用 NodePort Service 來 expose。 

17. 用 Node Selector 安排 Pod 到某個 Node。

> 其實 CKA 的考題真的比模擬考簡單的多，大家如果模擬考 OK 了就放心去考吧~


## 附錄：CKA 考試時常用的官方文件清單

CKA 考試時是可以查看官方文件的，這裡是我整理的清單，包含了「搜索關鍵字」與「說明文件」的對應，熟悉後就能快速找到相關說明：「[CKA 常用的官方文件](https://github.com/michaelchen1225/CKA-note/blob/main/%E9%99%84%E9%8C%84/CKA%20%E5%B8%B8%E7%94%A8%E7%9A%84%E5%AE%98%E6%96%B9%E6%96%87%E4%BB%B6.md)」。

> 結語：這次的鐵人賽花了 32 天才全部寫完，比預期的 30 天多了兩天，不過也算是順利完賽了，希望這次的系列文章有達到最初設定的目標：幫助到 k8s 的初學者與準備 CKA 的讀者！

---

**參考資料**

* [CKA 考試全攻略流程](https://medium.com/@app0/cka-%E8%80%83%E8%A9%A6%E5%85%A8%E6%94%BB%E7%95%A5%E6%B5%81%E7%A8%8B-3a28d1b73eea)

* [CKA證照考試心得整理(考前)](https://wade-software-study-note.medium.com/cka%E8%AD%89%E7%85%A7%E8%80%83%E8%A9%A6%E5%BF%83%E5%BE%97%E6%95%B4%E7%90%86-%E8%80%83%E5%89%8D-ba47b8562500)

* [我是如何一次通過CKA考試？不藏私的考試流程與經驗分享](https://medium.com/gemini-open-cloud/%E6%88%91%E6%98%AF%E5%A6%82%E4%BD%95%E4%B8%80%E6%AC%A1%E9%80%9A%E9%81%8Ecka%E8%80%83%E8%A9%A6-%E4%B8%8D%E8%97%8F%E7%A7%81%E7%9A%84%E8%80%83%E8%A9%A6%E6%B5%81%E7%A8%8B%E8%88%87%E7%B6%93%E9%A9%97%E5%88%86%E4%BA%AB-ebd65b33242a)

* [CKA Exam Tips: How To Crack The Exam In 2023 | Certified Kubernetes Administrator | KodeKloud](https://www.youtube.com/watch?v=8LJibrSurKA)
