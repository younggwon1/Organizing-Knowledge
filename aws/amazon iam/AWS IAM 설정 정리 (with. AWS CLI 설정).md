# AWS IAM 설정 정리 (with. AWS CLI 설정)

### 0. 준비사항

1. 사용하려고 하는 AWS 계정에서 IAM Role 및 MFA 가 설정되어 있어야 합니다. 
2. 또한, 로컬에 AWS CLI와 AWS-VAULT CLI가 설치되어 있어야 합니다.

**`awscli 설치`**
```bash
$ brew install awscli

$ aws --version
aws-cli/2.23.11 Python/3.12.8 Darwin/24.5.0 source/arm64
```

**`aws-vault 설치`**
```bash
$ brew install --cask aws-vault

$ aws-vault --version
v7.2.0
```

### 1. AWS IAM Role + MFA : 첫번째 방법
> IAM 사용자를 기반으로 IAM 역할 권한 사용하기

AWS에서는 사용자 계정의 권한을 직접 사용하는 대신 특정 IAM Role의 권한을 사용해서 작업할 수 있는 Assume Role(권한 위임) 기능을 지원하고 있습니다. 이 기능을 사용하면 IAM 사용자에게 직접 권한을 부여하지 않더라도 임시적으로 특정 역할을 권한을 사용할 수 있도록 하는 것이 가능합니다.

**설정방법**

~/.aws/ 경로에 config와 credentials 파일을 생성합니다.
```
$ mkdir ~/.aws
```

`~/.aws/credentials 파일`
```
$ aws-vault add {profile name}

ex)
$ aws-vault add dev-profile
Enter Access Key Id: ABDCDEFDASDASF
Enter Secret Key: %%%

$ aws-vault add prod-profile
Enter Access Key Id: ABDCDEFDASDASF
Enter Secret Key: %%%
```

```
[dev-profile]
aws_access_key_id = <YOUR_ACCESS_KEY_HERE>
aws_secret_access_key = <YOUR_SECRET_ACCESS_KEY_HERE>

[prod-profile]
aws_access_key_id = <YOUR_ACCESS_KEY_HERE>
aws_secret_access_key = <YOUR_SECRET_ACCESS_KEY_HERE>
```

`~/.aws/config 파일`
> 아래 ~/.aws/config는 샘플 설정파일이며, role_arn, mfa_serial 값들은 자신의 환경에 맞게 변경해서 사용하면 됩니다.

```
# ~/.aws/config
[default]
region = ap-northeast-2
output = json

[profile dev-profile]
role_arn = arn:aws:iam::111122223333:role/YYYYYYYYY
source_profile = dev-profile
region = ap-northeast-2
output = json
mfa_serial = arn:aws:iam::111122223333:mfa/XXXXXXXXX

[profile prod-profile]
role_arn = arn:aws:iam::111122225555:role/YYYYYYYYY
source_profile = prod-profile
region = ap-northeast-2
output = json
mfa_serial = arn:aws:iam::111122225555:mfa/XXXXXXXXX
```

IAM 정보를 확인하는 명령어
```
$ aws-vault exec {profile name} -- aws sts get-caller-identity

ex)
$ aws-vault exec dev-profile -- aws sts get-caller-identity

{
    "UserId": "XXXXXXXX:botocore-session-XXXXXXXX",
    "Account": "111122223333",
    "Arn": "arn:aws:sts::111122223333:assumed-role/XXXX/botocore-session-XXXXXXXX"
}
```
위와 같이 정상적으로 명령어 결과가 출력되어야 합니다.

이후 S3 의 리스트를 가져오는 AWS 명령어가 정상적으로 동작하는지 확인합니다.
```
$ aws-vault exec {profile name} -- aws s3 ls

ex)
$ aws-vault exec dev-profile -- aws s3 ls
```

설정 완료된 AWS CLI의 로컬 환경은 결과적으로 다음과 같습니다.

```
$ tree ~/.aws
/Users/<YOUR_USERNAME>/.aws
├── config
└── credentials
```

### 2. SSO(IAM Identity Center) : 두번째 방법

~/.aws/config 설정

```
$ aws configure sso
SSO session name (Recommended): {name}-sso
SSO start URL [None]: https://my-sso-portal.awsapps.com/start/#
SSO region [None]: ap-northeast-2
SSO registration scopes [None]: sso:account:access
```

웹에서 기기 및 인증 을 한 이후 ~/.aws/config 파일에 다음 profile 이 추가되었을 것입니다.

~/.aws/config 파일
> 아래 ~/.aws/config는 샘플 설정파일이며, role_arn, mfa_serial 값들은 자신의 환경에 맞게 변경해서 사용해야만 합니다.

```
# ~/.aws/config
[profile dev-sso]
sso_session = {name}-sso
sso_account_id = 111111111111
sso_role_name = {sso role name}
region = ap-northeast-2
output = json

[profile prod-sso]
sso_session = {name}-sso
sso_account_id = 2222222222222
sso_role_name = {sso role name}
region = ap-northeast-2
output = json

...

[sso-session {name}-sso]
sso_start_url = https://my-sso-portal.awsapps.com/start/#
sso_region = ap-northeast-2
sso_registration_scopes = sso:account:access
```

SSO Login
```
$ aws sso login --profile {profile name}

ex)
$ aws sso login --profile dev-sso
$ aws sso login --profile prod-sso
```

참고자료
- [AWS 공식문서](https://docs.aws.amazon.com/ko_kr/cli/latest/userguide/cli-configure-files.html)
- [aws-vault](https://github.com/99designs/aws-vault)


### 3. EKS 클러스터 등록
EKS 클러스터 등록
dev 환경
터미널에서 profile을 사용하도록 지정합니다.

```
$ export AWS_PROFILE={profile name}
``` 

SSO 로그인을 수행한 후 현재 자신의 IAM 권한 정보를 확인합니다.

aws sso login을 통해 로그인한 이후에, 다른 계정으로 전환할 때 다시 로그인할 필요가 없습니다. IAM Role에 선언된 세션 유지시간 동안은 다른 계정으로 다시 로그인할 필요 없이 프로필 전환만 하면 됩니다.

```
$ aws sso login
$ aws sts get-caller-identity
```

EKS 클러스터를 kubeconfig 파일에 등록합니다.

```
$ aws eks update-kubeconfig --region ap-northeast-2 --name {EKS Name}
```

로컬 환경에 등록된 EKS 클러스터 목록을 확인합니다.
```
$ kubectl config get-clusters
```