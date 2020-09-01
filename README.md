# Validate-MailServer-Configuration
Validate configuration about relay mail server in DMZ.





# 作成手順
## (1)事前設定
### (1)-(a) 作業環境の準備
下記を準備します。
* bashが利用可能な環境(LinuxやMacの環境)
* aws-cliのセットアップ
* AdministratorAccessポリシーが付与され実行可能な、aws-cliのProfileの設定

### (1)-(b) CLI実行用の事前準備
これ以降のAWS-CLIで共通で利用するパラメータを環境変数で設定しておきます。
```shell
export PROFILE=<設定したプロファイル名称を指定。デフォルトの場合はdefaultを設定>
export REGION=$(aws --profile ${PROFILE} configure get region)
```

## (2)VPCとPrivate Zoneの作成(CloudFormation利用)
IGWでインターネットアクセス可能で、パブリックアクセス可能なサブネットx2、プライベートなサブネットx2の合計4つのサブネットを所有するVPCを作成します。
<img src="./Documents/Step2.png" whdth=500>

### (2)-(a) Internal-VPC作成
ダウンロードしたテンプレートを利用し、VPCをデプロイします。
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "EnableNatGW",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcInternalDnsNameEnable",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "InternalVPC"
  },
  {
    "ParameterKey": "VpcCidr",
    "ParameterValue": "10.2.0.0/16"
  },
  {
    "ParameterKey": "PublicSubnet1Name",
    "ParameterValue": "FrontSubnet1"
  },
  {
    "ParameterKey": "PublicSubnet1Cidr",
    "ParameterValue": "10.2.0.0/19"
  },
  {
    "ParameterKey": "PublicSubnet2Name",
    "ParameterValue": "FrontSubnet2"
  },
  {
    "ParameterKey": "PublicSubnet2Cidr",
    "ParameterValue": "10.2.32.0/19"
  },
  {
    "ParameterKey": "PrivateSubnet1Name",
    "ParameterValue": "BackSubnet1"
  },
  {
    "ParameterKey": "PrivateSubnet1Cidr",
    "ParameterValue": "10.2.128.0/19"
  },
  {
    "ParameterKey": "PrivateSubnet2Name",
    "ParameterValue": "BackSubnet2"
  },
  {
    "ParameterKey": "PrivateSubnet2Cidr",
    "ParameterValue": "10.2.160.0/19"
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-InternalVPC \
    --template-body "file://./cfn/vpc-4subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;
```
### (2)-(b) DMZ-Inbound-VPC作成
ダウンロードしたテンプレートを利用し、VPCをデプロイします。
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableNatGW",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcInternalDnsNameEnable",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "DMZInboundVPC"
  },
  {
    "ParameterKey": "VpcCidr",
    "ParameterValue": "10.1.0.0/16"
  },
  {
    "ParameterKey": "PublicSubnet1Name",
    "ParameterValue": "PublicSubnet1"
  },
  {
    "ParameterKey": "PublicSubnet1Cidr",
    "ParameterValue": "10.1.0.0/19"
  },
  {
    "ParameterKey": "PublicSubnet2Name",
    "ParameterValue": "PublicSubnet2"
  },
  {
    "ParameterKey": "PublicSubnet2Cidr",
    "ParameterValue": "10.1.32.0/19"
  },
  {
    "ParameterKey": "PrivateSubnet1Name",
    "ParameterValue": "PrivateSubnet1"
  },
  {
    "ParameterKey": "PrivateSubnet1Cidr",
    "ParameterValue": "10.1.128.0/19"
  },
  {
    "ParameterKey": "PrivateSubnet2Name",
    "ParameterValue": "PrivateSubnet1"
  },
  {
    "ParameterKey": "PrivateSubnet2Cidr",
    "ParameterValue": "10.1.160.0/19"
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-DMZ-Inbound-VPC \
    --template-body "file://./cfn/vpc-4subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;
```

### (2)-(c) DMZ-Outbound-VPC作成
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "DnsHostnames",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "DnsSupport",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "InternetAccess",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "EnableNatGW",
    "ParameterValue": "true"
  },
  {
    "ParameterKey": "VpcInternalDnsNameEnable",
    "ParameterValue": "false"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "DMZOutboundVPC"
  },
  {
    "ParameterKey": "VpcCidr",
    "ParameterValue": "10.3.0.0/16"
  },
  {
    "ParameterKey": "PublicSubnet1Name",
    "ParameterValue": "PublicSubnet1"
  },
  {
    "ParameterKey": "PublicSubnet1Cidr",
    "ParameterValue": "10.3.0.0/19"
  },
  {
    "ParameterKey": "PublicSubnet2Name",
    "ParameterValue": "PublicSubnet2"
  },
  {
    "ParameterKey": "PublicSubnet2Cidr",
    "ParameterValue": "10.3.32.0/19"
  },
  {
    "ParameterKey": "PrivateSubnet1Name",
    "ParameterValue": "PrivateSubnet1"
  },
  {
    "ParameterKey": "PrivateSubnet1Cidr",
    "ParameterValue": "10.3.128.0/19"
  },
  {
    "ParameterKey": "PrivateSubnet2Name",
    "ParameterValue": "PrivateSubnet1"
  },
  {
    "ParameterKey": "PrivateSubnet2Cidr",
    "ParameterValue": "10.3.160.0/19"
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-DMZ-Outbound-VPC \
    --template-body "file://./cfn/vpc-4subnets.yaml" \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --capabilities CAPABILITY_IAM ;
```

### (2)-(d) Private Hosted Zone作成(CloudFormation利用)
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-PrivateHostedZone-VPC \
    --template-body "file://./cfn/privatehostedzone.yaml";
```
## (3)TransitGateway接続(CloudFormation利用)
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-DMZ-TGW \
    --template-body "file://./cfn/tgw.yaml" ;
```
## (4)VPCE作成(CloudFormation利用)
### (4)-(a) VPCE(PrivateLink)
```shell
# Internal-VPCへのVPCE作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "VpcStackName",
    "ParameterValue": "MailPoC-InternalVPC"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "InternalVPC"
  }
]'
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-Internal-VPC-VPCE \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/vpce.yaml" ;

# DMZ-Inbound-VPCへのVPCE作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "VpcStackName",
    "ParameterValue": "MailPoC-DMZ-Inbound-VPC"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "DMZInboundVPC"
  }
]'
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-DMZ-Inbound-VPC-VPCE \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/vpce.yaml" ;

# DMZ-Outbound-VPCへのVPCE作成
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "VpcStackName",
    "ParameterValue": "MailPoC-DMZ-Outbound-VPC"
  },
  {
    "ParameterKey": "VpcName",
    "ParameterValue": "DMZOutboundVPC"
  }
]'
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-DMZ-Outbound-VPC-VPCE \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/vpce.yaml" ;
```
### (4)-(b) VPCE(S3)
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-VPCES3 \
    --template-body "file://./cfn/vpce_s3.yaml" ;
```
## (5)SecurityGroup作成(CloudFormation利用)
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-SecurityGroups \
    --template-body "file://./cfn/sg.yaml" ;
```
## (6) バッチ・リレーメールインスタンスの準備
### (6)-(a) インスタンスロール作成 (CloudFormation利用)
```shell
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-IAM-For-Instance \
    --template-body "file://./cfn/iam-relaymail.yaml" \
    --capabilities CAPABILITY_NAMED_IAM ;
```
### (6)-(b)共通のパラメータ設定 
```shell
POSTFIX_MYNETWORK="127.0.0.1, 10.0.0.0/8"

KEYNAME="CHANGE_KEY_PAIR_NAME"  #環境に合わせてキーペア名を設定してください。  

#最新のAmazon Linux2のAMI IDを取得します。
AL2_AMIID=$(aws --profile ${PROFILE} --output text \
    ec2 describe-images \
        --owners amazon \
        --filters 'Name=name,Values=amzn2-ami-hvm-2.0.????????.?-x86_64-gp2' \
                  'Name=state,Values=available' \
        --query 'reverse(sort_by(Images, &CreationDate))[:1].ImageId' ) ;
echo -e "KEYNAME=${KEYNAME}\nAL2_AMIID=${AL2_AMIID}"
```
## (7)リレーメールインスタンス
### (6)-(a) (Option)Gmail接続用のパスワードのStore
検証でGmailに接続するためのユーザIDとパスワードを安全に管理するため、System ManagerのParameter storeに格納します。
```shell
# GMAILの接続用の認証情報の設定
# username@gmail:passwordに、アカウント名とgoogleで払い出したアプリパスワードを設定します。
GMAIL_SASL_INFO="[smtp.gmail.com]:587 username@gmail:password"

# Parameter storeのPut
aws --profile ${PROFILE} \
  ssm put-parameter \
    --name "mail-poc-gmail-sals-info" \
    --value "${GMAIL_SASL_INFO}" \
    --type SecureString \
    --tags "Key=Name,Value=mail-poc-gmail-sals-info"
```

### (7)-(b) OutboundのRelayMailインスタンス作成 (CloudFormation利用)
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "KeyName",
    "ParameterValue": "'"${KEYNAME}"'"
  },
  {
    "ParameterKey": "AmiId",
    "ParameterValue": "'"${AL2_AMIID}"'"
  },
  {
    "ParameterKey": "PostfixMynetwork",
    "ParameterValue": "'"${POSTFIX_MYNETWORK}"'"
  }
]'

aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-Outbound-RelayMail-Instance \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/relaymail.yaml" \
    --capabilities CAPABILITY_NAMED_IAM ;
```

## (8) バッチインスタンス作成(CloudFormation利用)
```shell
CFN_STACK_PARAMETERS='
[
  {
    "ParameterKey": "KeyName",
    "ParameterValue": "'"${KEYNAME}"'"
  },
  {
    "ParameterKey": "AmiId",
    "ParameterValue": "'"${AL2_AMIID}"'"
  },
  {
    "ParameterKey": "PostfixMynetwork",
    "ParameterValue": "'"${POSTFIX_MYNETWORK}"'"
  }
]'
aws --profile ${PROFILE} cloudformation create-stack \
    --stack-name MailPoC-Batch-Instance \
    --parameters "${CFN_STACK_PARAMETERS}" \
    --template-body "file://./cfn/batch.yaml" \
    --capabilities CAPABILITY_NAMED_IAM ;
```
## (9) テスト
### (9)-(a)メール送信テスト
```shell
Subject="TestMail-$(date '+%Y%m%d%H%M%S')"
From_Address="xxx"
To_Address="nobuyuf@amazon.co.jp"

echo "テストメール$(date '+%Y%m%d%H%M%S')" | mail -s $Subject -r $From_Address $To_Address
```