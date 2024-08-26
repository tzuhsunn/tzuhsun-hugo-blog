---
title: "理解 Kubelet Logs (翻譯+筆記)"
date: 2024-08-26T02:30:35+08:00
lastmod: 
categories: 
- Kubernetes
tags: 
- Kubernetes
- Kubelet
- disaster
summary: "理解 Kubelet 冗長日誌"
description: "理解 Kubelet 冗長日誌" 
weight: 
slug: ""
draft: false 
hidemeta: false
cover:
    image: "" 
    caption: ""
    alt: ""
    relative: false
---
# 說明
原文：[Navigating the Everest of Logs \- A Guide to Understanding Kubelet Logs](https://kodekloud.com/blog/navigating-the-everest-of-logs-a-guide-to-understanding-kubelet-logs/?utm\_source=udemy\&utm\_medium=email\&utm\_campaign=educational-email)  
你可能看過這個有趣的網路迷因(meme),幽默地展示當某些事情出錯時,Kubelet日誌是多麼令人眼花撩亂。雖然看到這些冗長的日誌很有趣,但實際上它們確實可能相當複雜。然而,這座資訊山脈其實非常有用,如果你能夠理解它們,將有助於你修復Kubernetes叢集中的問題。本指南將成為你開始理解這些冗長日誌的起點,為你提供一些技巧和訣竅,幫助你輕鬆導覽這些日誌。
{{< figure  src="meme.png" title="kubelet-meme" width="70%" align="center">}}

## What is Kubelet…

在深入探討Kubelet日誌之前,讓我們先了解什麼是**Kubelet**以及它的作用。Kubelet運行於Kubernetes叢集中的每個***節點***上。 

*節點是運行應用程式的個別電腦或伺服器。有兩種類型的節點:控制平面節點(control plane nodes, 管理整個叢集)和工作節點(worker nodes, 運行應用程式工作負載)。*

Kubelet與control plane進行通訊。control plane對cluster作出決策,例如調度工作負載,並存儲叢集狀態。Kubelet從控制平面接收指令,並回報節點和其Pod的狀態。 Kubelet的主要工作是確保容器在Pod中運行。 

*Pod是Kubernetes中最小、最基本的單位,代表一個應用程式實例。* 

Kubelet持續監控節點和其Pod的健康狀況。它執行定期檢查,並向control plane 報告任何問題。如果Pod或節點不健康,Kubelet會嘗試重新啟動Pod或採取其他糾正措施。 此外,Kubelet從control plane讀取Pod規格,並確保使用指定配置運行正確的容器。如果配置有任何變更,它也會更新正在運行的Pod。

## Understanding Kubelet Logs

Kubelet日誌是由運行在Kubernetes集群中每個節點上的Kubelet agent 所創建的記錄。這些日誌對於診斷問題和確保容器在其Pod中順利運行至關重要。但它們可能相當詳細和複雜,因此導覽這些日誌很具挑戰性。

### 為什麼我們需要Kubelet日誌?

Kubelet日誌出於幾個原因而至關重要。它們有助於通過識別和解決Kubernetes集群內的問題來診斷問題。透過理解這些日誌,您可以監控應用程序的性能,確保它們運行順暢。日誌還在security auditing中扮演著重要角色,通過tracing cluster內的訪問和操作來維護安全性。此外,保留詳細的系統活動記錄對於滿足監管合規性要求也是必不可少的(meeting regulatory compliance requirements)。這些歷史資料可確保您對集群內發生的事件有清晰的記錄。

### 為什麼Kubelet日誌如此冗長?

* 容器啟動和關閉事件: 每當容器啟動或停止時,都會記錄下來。這有助於您跟踪容器發生的情況,並了解任何問題的根源。  
* 健康檢查: Kubelet定期檢查容器和節點的健康狀況。這些檢查會被記錄下來,幫助您查看出現錯誤的時間和原因。  
* 節點狀態更新: Kubelet記錄每個節點的狀態,例如它是否準備就緒可以運行pod,或者是否存在任何問題。這些更新為節點的健康狀況提供了一個完整的時間線視圖。  
* 與API服務器的互動: Kubelet與Kubernetes API服務器溝通以獲取指令並回饋狀態。這些互動都被記錄下來,幫助了解流程並發現任何問題。  
* 錯誤和警告: Kubelet遇到的任何錯誤或警告都會被記錄下來。這些消息對於修復問題和保持集群順利運行至關重要。

## See Kubelet Logs

Kubelet將其活動記錄到systemd日誌中。通過將這些日誌存儲在systemd日誌中,可以使用systemd工具輕鬆訪問和管理它們。

現在我們知道日誌存儲在哪裡了。那麼如何訪問它們呢?這就是`journalctl`命令的用武之地。`journalctl`是一個命令行工具,用於查詢和顯示來自**systemd日誌**的日誌。它允許您過濾和搜索日誌,從而更容易找到特定信息。這對於訪問Kubelet日誌和對Kubernetes集群進行故障排除非常有用。

使用`journalctl`,您可以看到systemd日誌收集到的所有日誌。例如,運行`journalctl -b`可顯示自上次系統啟動以來的日誌。這有助於理解節點上的最新活動。

要專注於Kubelet日誌,您可以使用`-u`標誌,後跟`kubelet`。命令`journalctl -u kubelet`將顯示與Kubelet服務相關的所有日誌,從而更容易診斷問題。

如果希望kubelet的logs 可以更加詳細，修改kubelet的設定檔並重啟。

1. 編輯kubeadm.conf文件   
2. 修改`KUBELET_CONFIG_ARGS`以包含`--v=5`。  
3. 保存文件並重新啟動Kubelet。