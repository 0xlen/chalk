---
title: "身為 DevOps 你會想知道的 AWS 技巧 - 使用 AWS Lambda 和 Amazon SNS 取得來自 AWS CodeCommit (Git) 的檔案變更通知"
description: "在這篇內容中，我會展示如何整合 AWS Lambda 及 Amazon SNS 以幫助你擴展 AWS CodeCommit 的功能性"
tags: ['aws', 'amazon web services', 'Lambda', 'Lambda Function', 'severless', 'CodeCommit', 'AWS CodeCommit', 'SNS', 'Amazon SNS', 'Git']
header:
  og_image: /assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/architecture.png
  teaser: /assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/architecture.png
toc: true
toc_label: "Table of Contents"
toc_icon: "cog"
classes: wide
---

這篇內容主要是轉載我 2019 年 2 月公開發佈在 AWS 官方 DevOps 部落格的內容，在這篇內容中，主要展示了如何以 AWS Lambda 及 Amazon SNS 接收來自 AWS CodeCommit (Git Repository) 的檔案變更通知。由於屆時將滿 2 年 (時間過得真快)，覺得有必要翻譯成中文文件，以幫助中文的讀者也能夠了解這項內容。

由於原內容為英文，如果有興趣，原文請見：

- [Using AWS Lambda and Amazon SNS to Get File Change Notifications from AWS CodeCommit](https://aws.amazon.com/blogs/devops/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/)。


## 簡介

通知一直是 DevOps 工作流程中很重要的一環，幾乎很多牽扯維運相關的工作都少不了主動通知的行為。當然，你可以在任何 CI/CD 的階段中透過你已知的方法於任何邏輯中設置。但在這篇 Blog Post 中，我將會展示如何整合 AWS Lambda 和 Amazon SNS 以擴展 AWS CodeCommit 的功能性。特別的是，這篇解決方內容案描述了當 AWS CodeCommit 一旦有任何更新，如何從 Amazon SNS 中接收檔案變更和 Commit 訊息。以下主要簡介相應使用到的 AWS 服務：

- [Amazon SNS](https://aws.amazon.com/sns/): Amazon Simple Notification Service (Amazon SNS) 是一項全受管簡訊服務，並且能夠幫助你發佈訊息至訂閱者。其非常容易使用並且支援任何規模大小的使用場景。

- [AWS Lambda](https://aws.amazon.com/lambda/): 是一種無伺服器的運算服務，可讓您執行程式但不必佈建或管理伺服器、建立工作負載感知叢集擴展邏輯、維護事件整合或管理執行階段。使用 Lambda，您可以透過虛擬方式執行任何類型的應用程式或後端服務，全部無需管理。在這篇內容中，我使用了 Lambda Function 以推送訊息至 Amazon SNS 以發佈檔案更新。
Amazon CloudWatch

- [Amazon CloudWatch](https://aws.amazon.com/cloudwatch) 提供資料和可行的洞見以監控應用程式、回應整個系統的效能變化、優化資源使用情況，以及透過整合的檢視來查看運作狀態。[你可以也設定簡易的規則來偵測 AWS 資源的變更](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/Create-CloudWatch-Events-Rule.html)。在 CloudWatch 捕捉到來自你 AWS 資源的事件更新後，便能觸發特定的目標執行相應的操作 (例如：觸發一個 Lambda Function)


為幫助你快入了解並且部署這項解決方案，我同時建立了一個 AWS CloudFormation template 以供這篇內容使用。[AWS CloudFormation](https://aws.amazon.com/cloudformation/) 是一個管理工具並且能夠使用通用的語法幫助你描述並且部署 AWS 相關的基礎建設和資源。

## 概覽 (Overview)

{% include figure image_path="/assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/architecture.png" alt="架構概覽" caption="架構概覽" %}

AWS CodeCommit 支持了多項實用的 [CloudWatch Event](https://docs.aws.amazon.com/AmazonCloudWatch/latest/events/EventTypes.html#codecommit_event_type)，透過 CloudWatch Event，這能夠幫助你監控 AWS 資源的使用事件變更並且進行通知。透過設定一些簡單的觸發規則，你便能夠偵測有關 branch 或是 Repository 的變更。

在這個範例中，我為一個 AWS CodeCommit Repository 建立了一個 CloudWatch event rule (事件規則)，如此一來，任何相應的變更事件便會觸發一個 Lambda Function。當對於 CodeCommit 的變更執行時，CloudWatch 便偵測該事件並且執行自定義 Lambda Function 的觸發。

當這個 Lambda Function 被觸發，下列的行為將被依序執行：

- 使用 CodeCommit API 中的 [GetCommit](https://docs.aws.amazon.com/codecommit/latest/APIReference/API_GetCommit.html) 操作取得最後一次的 Commit 紀錄。因為我想要比較原先上一次的 Commit ID 和最新的一筆，以進行變更的檢查。
- 對每一個 Commit 紀錄，使用 [GetDifferences](https://docs.aws.amazon.com/codecommit/latest/APIReference/API_GetDifferences.html) API 操作取得任何紀錄追蹤檔案的新增、變更和刪除資訊。
- 從比較的結果中，合併相關的變更資訊，並且，將這項資訊依照定義好的 Email 訊息格式推送到 Lambda Function 中環境變數中定義的 Amazon SNS (SNS topic) 資源。
- 允許 Reviewers (可能是 Code Reviewers, operation team ... 等) 訂閱該 SNS Topic。如此一來，任何有關 CodeCommit 相應的更新都能推播到相關的訂閱者。

這個範例使用了 Python 和 Boto3 實作這項功能。完整的程式碼已被公開在 GitHub 上，你可以在 AWS 官方的 GitHub 上找到 - [aws-codecommit-file-change-publisher](https://github.com/aws-samples/aws-codecommit-file-change-publisher) 該範例。


## 開始動手做

為了幫助你快速搭建這項解決方案，該專案包含了一個 AWS CloudFormation template (`codecommit-sns-publisher.yml`)。這個 template 使用了 [AWS Serverless Application Model (AWS SAM)](https://github.com/awslabs/serverless-application-model) 用以定義 CodeCommit 通知 Serverless 應用程式必須的組建，並且使用簡潔的語法表示。

一旦部署後，這個 template 會被直接轉譯成 AWS CloudFormation stack 資源並且建立一個 SNS Topic, CloudWatch event rule 和一個 Lambda Function。這個 Lambda Function 程式碼已經展示了一個簡易的通知功能用例。你可以使用這項範例程式自行定義你的邏輯，甚至是使用其他 [AWS SDK for Python (Boto3)](https://aws.amazon.com/sdk-for-python/) 提供的 API 和方法擴展功能。

### 預先準備工作 

在開始部署該範例之前，你必須先[建立/擁有一個 CodeCommit repository](https://docs.aws.amazon.com/codecommit/latest/userguide/how-to-create-repository.html)，以便後續使用 AWS CloudFormation template 執行相應的操作。在這篇文章中，我在 Ohio 區域 (us-east-2) 建立了一個乾淨的 CodeCommit Repository (`sample-repo`) 以便展示一個 CodeCommit Repository 在特定 branch 上有檔案修改更新。

如果你已經擁有一個 CodeCommit Repository，你可以繼續往下閱讀下列步驟以部署 template 和 Lambda Function。

### 部署 AWS CloudFormation template 及 Lambda function

- 下載 [aws-codecommit-file-change-publisher](https://github.com/aws-samples/aws-codecommit-file-change-publisher) 上的原始碼

- 登入至 AWS Console 及 [選擇你 CodeCommit Repository](https://docs.aws.amazon.com/awsconsolehelpdocs/latest/gsg/getting-started.html#select-region) 所在的區域。並且，手動建立一個 S3 Bucket 並且上傳 AWS Lambda deployment package (封裝好的 zip 文件 - `codecommit-sns-publisher.zip`)。如果你不確定如何建立 S3 Bucket，請參考 Amazon S3 Console 的操作手冊 - [How Do I Create an S3 Bucket?](https://docs.aws.amazon.com/AmazonS3/latest/user-guide/create-bucket.html) 以引導你完成

- 上傳 Lambda deployment package 至你剛剛建立好的 S3 Bucket

在這個範例中，我在相同區域 (Ohio, us-east-2) 建立了一個 S3 Bucket 名為 `codecommit-sns-publisher` 並且透過 Amazon S3 上傳 Lambda deployment package：

{% include figure image_path="/assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/upload-lambda-package-to-s3-bucket.png" alt="上傳 Lambda deployment package 至 S3" caption="上傳 Lambda deployment package 至 S3" %}

- 在 AWS Management Console, 選擇 CloudFormation 導引到該服務。你也可以直接使用這個連結訪問 - [https://console.aws.amazon.com/cloudformation](https://console.aws.amazon.com/cloudformation)

- 選擇 **Create Stack**

- 在 **Select template** 頁面，選擇 **Upload a template to Amazon S3**，指定剛剛下載的 `codecommit-sns-publisher.yml` template 檔案上傳進行下一步

{% include figure image_path="/assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/upload-cfn-template.png" alt="選擇 CloudFormation template" caption="選擇 CloudFormation template" %}

- 在指定參數項目中，填寫以下資訊：
  - **Stack Name**: codecommit-sns-publisher (你可以指定自己的名稱)
  - **CodeS3BucketLocation**: `codecommit-sns-publisher` (指定你剛剛在建立 S3 Bucket 步驟中上傳 Lambda deployment package 的 S3 Bucket 名稱)
  - **CodeS3KeyLocation**: `codecommit-sns-publisher.zip` (這是上傳 Lambda deployment package 的名稱, 物件應為 zip 檔案) 
  - **CodeCommitRepo**: `sample-repo` (你 CodeCommit repository 的名稱)
  - **MainBranchName**: `master` (指定你想觸發事件的 branch 名稱)
  - **NotificationEmailAddress**: `user@example.com` (指定要訂閱 SNS topic 的 Email 地址, 這個設置可以讓 CloudFormation template 建立一個 SNS topic 以推送通至訂閱者)


{% include figure image_path="/assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/cfn-typing-parameters.png" alt="指定參數 Parameters" caption="指定參數 Parameters" %}


- 選擇 **Next**

- 在 **Review** 頁面中，於 **Capabilities** 項目底下，勾選以下選項：
  - I acknowledge that AWS CloudFormation might create IAM resources.
  - I acknowledge that AWS CloudFormation might create IAM resources with custom names.

{% include figure image_path="/assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/cfn-create-change-set-acknowledgement.png" alt="選擇 Capabilities" caption="選擇 Capabilities" %}

- 在 **Transforms** 項目，點擊 **Create Change Set**。AWS CloudFormation 便會開始執行 template 轉譯的工作並且建立一個 Change Set

{% include figure image_path="/assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/cfn-create-change-set-transforms.png" alt="建立 Change Set" caption="建立 Change Set" %}

- 在完成轉譯後，選擇 **Execute** 以建立 AWS CloudFormation stack

{% include figure image_path="/assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/preview-change-set.png" alt="執行建立 CloudFormation Stack" caption="執行建立 CloudFormation Stack" %}

在 Stack 建立完成後，若資源有正確建立，你應該可以預期會在信箱收到 SNS 訂閱確認信，請點擊確認，看到以下內容即成功訂閱：

{% include figure image_path="/assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/sns-subscribe.png" alt="SNS 訂閱確認成功訊息" caption="SNS 訂閱確認成功訊息" %}


在你訂閱了 SNS Topic 之後，你便可以在 AWS CloudFormation Console 檢查對應建立出來的資源。如果你想要監控 Lambda Function 的執行狀態，點擊 **Resources** 可以開啟 Lambda Function(`SNSPublisherFunction`)：

{% include figure image_path="/assets/images/posts/2021/02/using-aws-lambda-and-amazon-sns-to-get-file-change-notifications-from-aws-codecommit/create-cfn-stackCWE.png" alt="CloudFormation 部署的資源" caption="CloudFormation 部署的資源" %}


## 現在，你可以在本機嘗試推送一個 Commit 至遠端的 AWS CodeCommit Repository


Step 1. 下載你的 (git clone) CodeCommit repository 至你的本機電腦。更多有關連接到 AWS CodeCommit 和驗證的資訊，請參考 AWS CodeCommit 使用手冊中的 [Connect to an AWS CodeCommit Repository](https://docs.aws.amazon.com/en_us/codecommit/latest/userguide/how-to-connect.html) 內容幫助你設定。在這個範例中，展示了如何下載位於 Ohio 區域 (us-east-2) 名為 `sample-repo` 的 repository：

```bash
git clone ssh://git-codecommit.us-east-2.amazonaws.com/v1/repos/sample-repo
```

Step 2. 進入至該專案目錄並且見一粒一個純問自檔案

```bash
cd sample-repo/
echo 'This is a sample file' > newfile
```

Step 3. 新增一個 Commit 並且紀錄這次的修改

```bash
git add newfile
git commit -m 'Create initial file'
```

[輸出]

```
[master (root-commit) 810d192] Create initial file
1 file changed, 1 insertion(+)
create mode 100644 newfile
```

Step 4. 推送到遠端的 CodeCommit Repository

```bash
git push -u origin master:master
```

[輸出]

```
Counting objects: 100% (3/3), done.
Writing objects: 100% (3/3), 235 bytes | 235.00 KiB/s, done.
…
* [new branch]      master -> master
Branch 'master' set up to track remote branch 'master' from 'origin'.
```

在本機的 Commit 更改推送到遠端 CodeCommit Repository 後，將會觸發 CloudWatch event 並且偵測到這次的更新。你通常可以在訂閱的 Email 帳戶中預期看到以下的通知訊息：

```
Commit ID: <Commit ID>
author: [YourName] (YourEmail@example.com) - <Timestamp> +0000
message: Create initial file

File: newfile Addition - Blob ID: <Blob ID>
```


## 總結

在這篇內容中，我展示了如何使用 AWS CloudFormation template 快速部署一個範例的解決方案，能夠幫助你的維運或是開發團隊追蹤任何有關 CodeCommit Repository 的更新。本篇示例的 CloudFormation template 及 Lambda Function 同時也在 AWS 官方的 GitHub 上被公開 - [aws-codecommit-file-change-publisher](https://github.com/aws-samples/aws-codecommit-file-change-publisher)。你可以依據你的需求參考這個範例程式幫助你自定義 Email 的內容 (例如加上 HTML 樣式)，亦或者是新增其他有用的訊息至你的 Email 訊息中。

若你對於這項實作感到興趣或是有其他建議，也歡迎在底下留言與我分享。當然，如果有任何關於開源專案的反饋，也歡迎開啟 GitHub issue 甚至是開啟 GitHub pull request 貢獻！
