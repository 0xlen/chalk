---
title: "[Alexa] 靠一張嘴巴維運，我是如何成為出一張嘴的雲端工程師 - 使用 Alexa skill 管理 AWS EC2 防火牆 (Security Group) 規則"
description: "Using Alexa Skill to help you manage website firewall (Security group of EC2 instance)"
tags: ['aws', 'amazon web services', 'EC2', 'Elastic Compute Cloud', 'amazon', 'alexa', 'alexa skill', 'echo', 'echo dot']
header:
  og_image: /assets/images/posts/2020/08/security-group-manager-alexa-skill/security-group-manager-alexa-skill-architecture.png
---

隨著 COVID-19 疫情大爆發，科技圈再次掀起了一股遠端辦公的趨勢，遠距工作的形式也逐漸改變許多人的工作型態。身為一名工程師，因應疫情，今年也頻繁的在家辦公也好幾個月了 (還好台灣還很安全)，仍有很多與傳統地域限制型態的工作模式有很多不同的地方。

平常在辦公室，由於直接連接公司內部網路，通常都有特定的 IP 區段可以很簡單的掌握白名單，防火牆規則都非常好設置。但是自從開始遠端工作後，家裡的網路都是使用浮動 IP，有時候工程師的惰性驅使，又很懶得連上 VPN 在那邊拿 MFA Key  驗證身份，光是要設定自己 EC2 資源的白名單常常都要改來改去的。與此同時，在日常工作中也偶爾跟同事屁來屁去講一些垃圾話，注意到一些覺得可以進行自動化、增加生產力的一些想法及提案，秉持著發明家的精神，於是以下我就來說說我是如何靠一張嘴巴滿足我的懶惰的。

## 簡介

在沒有靠嘴巴維運之前，為了要正確的更改防火牆規則讓我能夠在家中工作時能夠連上跳板機器操作，我是這麼做的：

- 打開 AWS Management Console
- 輸入帳號密碼
- 彈出 MFA (Multi-Factor Authentication) 認證 -> 打開手機 App 確認現在的動態碼 -> 輸入動態碼
- 打開 EC2 Console
- 選擇 EC2 / Security Group
- 更新 EC2 規則 (獲取自己的外部 IP 地址更新上去)

上述的動作常常都要花我 3-5 分鐘，有時候手機丟在臥室不在身邊又要走超遠！為了滿足我的懶惰，以下是我目前的工作流程，先來看一段示範影片

{% include video id="3AFGvt1Ck5c" provider="youtube" %}

上面的動作，只要正確的發出聲音指令後，Alexa Skill 便可以正確地更新相應的 Security Group 規則 (並且需要符合自己定義的 PIN Code，避免所有人都可以任意的發出命令更改)，並且只開放我家中正確對外的特定 IP 地址。如此一來，我就能直接地在家透過 SSH 連接上對應的 EC2 Instance 進行工作。

## 什麼是 Amazon Alexa ?

大概就像是 Apple 的 Siri、Google 推出的 Google Home ...

> Amazon Alexa，簡稱 Alexa，是亞馬遜公司推出的一款智能助理，最初用於 Amazon Echo 智能音箱。它具有語音交互、音樂播放、待辦事項列表、鬧鐘、流播播客、播放有聲讀物以及提供天氣，交通，體育和其他實時信息（如新聞）的功能[3]。Alexa還可以將自身用作智慧家庭系統來控制多個智能設備。該產品由Amazon Lab126開發，是一名女性語音助手。用戶可以通過安裝插件（由第三方供應商開發的其他功能）來擴展Alexa功能。

> Alexa的大多數設備允許用戶通過一個特定的詞語（如「Alexa」或「Amazon」）來喚醒，剩下的（如IOS和Andriod上的Amazon移動應用與Amazon Dash Wand）則需用戶通過按按鈕來使之進入聆聽模式。一些其他廠商的手機也同樣支持用戶發出一些指令來喚醒屏幕，如「Alexa」或「Alexa wake」。到目前為止，Alexa的交流和應答僅可用英語、德語、法語、義大利語、西班牙語、葡萄牙語、日語和印地語。Alexa在加拿大可用於英語和法語（包括魁北克法語）。

來源: [Wikipedia](https://zh.wikipedia.org/wiki/Amazon_Alexa)

## 架構概覽 (Architecture Overview)

{% include figure image_path="/assets/images/posts/2020/08/security-group-manager-alexa-skill/security-group-manager-alexa-skill-architecture.png" alt="Alexa Skill - Security Group Manager 架構概覽" caption="Alexa Skill - Security Group Manager 架構概覽" %}

上述的架構在我看完 Alexa Skills Kit SDK for Python 的範例後，整體的實作到部署我大概花了 3 - 5 小時實作出來。目前含註解整個代碼不超過 300 行，難度並不會太高，以下具體描述一些實作細節。

## 實作細節

### 怎麼開發？

照著 [Alexa Skills Kit SDK for Python](https://github.com/alexa/alexa-skills-kit-sdk-for-python) 快速實作了一個可以動的版本。如果你是第一次玩 Alexa Skill，[Color Picker](https://github.com/alexa/skill-sample-python-colorpicker/blob/master/instructions/1-voice-user-interface.md) 是一個不錯的範例，你可以找到很多不同語言的相應實做。

### 怎麼上傳到 AWS Lambda

建立完 AWS Lambda Function 後，便可以打包你的程式碼上傳到 Lambda 執行。

一般來說，AWS 文件可能會建議你使用 VirtualEnv 在本機建立 (如同在之前 [使用 AWS Lambda 建立 Line bot](https://easoncao.com/create-a-line-bot/) 中提到的)，但我其實偷了一些懶，直接用 Docker image 加上一行 Docker 命令打包 deployment package (python):

```bash
docker run -v "$PWD":/var/task "lambci/lambda:build-python3.6" /bin/sh -c "pip install -r requirements.txt -t package/; cp lambda_function.py package/; cd package; zip -r9 lambda.zip ."
```

過程中會生成 `package/` 目錄，上傳該目錄底下壓縮完的 `lambda.zip` 即可。

### 安全性設置 - IAM Policy

我在 Lambda Function 使用的 Execution Role 中定義了下列規則：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "UpdateSG",
            "Effect": "Allow",
            "Action": [
                "ec2:RevokeSecurityGroupIngress",
                "ec2:AuthorizeSecurityGroupEgress",
                "ec2:AuthorizeSecurityGroupIngress",
                "ec2:RevokeSecurityGroupEgress"
            ],
            "Resource": "arn:aws:ec2:<REGION>:<ACCOUNT_ID>:security-group/sg-XXXXXXXX"
        },
        {
            "Sid": "DescribeSG",
            "Effect": "Allow",
            "Action": "ec2:DescribeSecurityGroups",
            "Resource": "*"
        }
    ]
}
```

如此一來可以避免我的應用程式未經授權操作其他的 Security Group 資源，並且只允許更改特定 Security Group ID 的相關 入站(Ingres) / 出站(Egress) 規則。

### Alexa 如何接受指令？

在 [Amazon Alexa Developer Console](https://developer.amazon.com/alexa/console/ask) 中建立完 Alexa Skill 後，可以進行相關的定義。 Alexa 會根據 Interaction Model 設定很多不同的 Intent，每個 Intent 能夠對應的相應操作行為，以下是一個當我想要輸入 PIN code 各種不同的命令範例：

{% include figure image_path="/assets/images/posts/2020/08/security-group-manager-alexa-skill/alexa-skill-intents-enter-pin-example.png" alt="Alexa Skill - Security Group Manager 設置 Intent" caption="Alexa Skill - Intent 設置" %}

如此一來，在應用程式中便可以使用動態的名稱 `EnterPINCodeIntent` 和變數 `PIN_CODE` 來執行並且獲取一些邏輯。

### Lambda 如何更新 Security Group?

當 Alexa 根據觸發 Lambda 的時候，例如：`Alexa, update my security group!`，根據我的設置，便觸發 "UpdateMySGIntent"，於是在相關的執行邏輯在包含 PIN Code 驗證的情況，可以是以下：

```python
@sb.request_handler(can_handle_func=is_intent_name("UpdateMySGIntent"))
def update_my_sg_handler(handler_input):
    """Check if PIN code is provided in session attributes alues. If provided, then
    update security group with invoking source IP address.
    If not, then it asks user to provide the PIN code.
    """
    # type: (HandlerInput) -> Response
    slots = handler_input.request_envelope.request.intent.slots

    if PIN_CODE_SLOT_KEY in handler_input.attributes_manager.session_attributes and handler_input.attributes_manager.session_attributes[PIN_CODE_SLOT_KEY] == pre_defined_pin_code:
        speech = ("PIN code matches, I am updating the security group.")
        handler_input.response_builder.speak(speech)

        update_response = update_security_group()

        speech = ("Security group has been updated. {}").format(update_response)
        reprompt = ("You can ask me your security group setting by saying, "
                    "what's my security group ?")
    else:
        speech = "You did not correctly specify the PIN code or the PIN code doesn't match"
        reprompt = ("I'm not sure what your PIN code is, "
                    "You can say, "
                    "my PIN code is blah blah blah....")

    handler_input.response_builder.speak(speech).ask(reprompt)
    return handler_input.response_builder.response
```

在 Python 應用程式中，使用 DDNS 解析獲取目前外部 IP 後，更新 Security Group 的行為主要使用了 EC2 - [AuthorizeSecurityGroupIngress API](https://docs.aws.amazon.com/AWSEC2/latest/APIReference/API_AuthorizeSecurityGroupIngress.html) 執行了這項操作：

```python
def update_security_group():
    ec2 = boto3.resource('ec2', region_name=security_group_region)
    security_group = ec2.SecurityGroup(security_group_id)
    source_ip = get_source_ip()
    cidr_ip = source_ip + '/32'

    security_group.authorize_ingress(IpProtocol="tcp",CidrIp=cidr_ip,FromPort=22,ToPort=22)

    response = ("The address {} has been added to the security group {} in region {}").format(source_ip, security_group_id, security_group_region)

    print(response)

    return response
```

### Alexa Skill 如何觸發 Lambda?

在 [Amazon Alexa Developer Console](https://developer.amazon.com/alexa/console/ask) 中可以設置自定義的 Lambda function endpoint：

{% include figure image_path="/assets/images/posts/2020/08/security-group-manager-alexa-skill/alexa-skill-setup-lambda-endpoint.png" alt="Alexa Skill - Security Group Manager 設置 Lambda endpoint" caption="Alexa Skill - 設置 Lambda endpoint" %}

### Lambda 如何知道目前的使用者外部來源 IP

根據我的觀察，由於 Alexa 觸發 Lambda Function 時，是由 Voice Server 去戳 Lambda，並且也沒有附帶相關的 IP 訊息。因此，為了達成我的目的，我主要使用了如同架構中描述的方式獲取 DDNS 中的紀錄取得真實外部 IP 地址後，進行更改。

透過路由器韌體通常都支援更改 DDNS 的設定，通常好一點的路由器都支持一些很奇耙的功能，如果你會設定的話可以完成蠻多有趣的事情 (例如：[在中華電信提供的 D-Link DSL-7740C 啟用 SNMP](https://easoncao.com/enable-snmp-agent-on-dlink-dsl-7740c/))。我的網路環境使用了 D-Link DSL-7740C，透過 D-Link 韌體提供的功能，能夠輕鬆的設定並且直接幫助更新我的 Dynamic DNS record，再由 Lambda Function 邏輯中主動解析獲取。

同理，另一種方式是你可以在你的環境中運行一隻小程式 (agent) 或是透過 Cronjob 定義的 Shell Script，幫助你更新 DNS 紀錄，也是一樣的方法。

若你知道 [API Gateway](https://aws.amazon.com/tw/api-gateway/) 是什麼的話，也許可以利用 API Gateway 搭建 endpoint 並且在上個動作的階段改用 Custom HTTPS endpoint 觸發，或許在交付客戶端的 Alexa device (比如 Echo dot) 觸發時，是由客戶端主動進行訪問，這種情況下，在附帶的請求訊息中也許能知道相關的 IP，但是我沒試過，如果你試了，也歡迎在底下分享你的發現。

## 總結

本篇內容簡介了一項使用 Alexa Skill 進行系統維運的實作方法，展示了工程師的懶惰成性，並且分享一項參考架構，提及如何靠一張嘴巴更改防火牆規則，同時提及相關的實作細節。透過聲音命令簡化並且自動化繁瑣的更新步驟，減少了每次花費 3-5 分鐘的操作時間 (操作 10 次省下 30 分鐘、100次省下 300 分鐘 ... 請自行誇飾及想像)，幫助提升日常工作中的相應生產力。

若你對於這項實作感到興趣或是有其他建議，也歡迎在底下留言與我分享！
