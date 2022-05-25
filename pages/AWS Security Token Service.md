- 文档： https://docs.aws.amazon.com/STS/latest/APIReference/welcome.html
-
- STS 一般是用来跟 IAM 配合使用，用来生成一个临时的，有限制的 Token 给应用使用。
-
- 比较常用的是 `AssumeRoleWithWebIdentity`
  collapsed:: true
	- 通过 [[AWS IAM]] 分配的 WenIdentityToken 来向 STS 换取 Token
	- Sample Request
		- STS 服务使用 Query 签名，认证信息都在 query 中
			- 这里的 `RoleArn` 和 `WebIdentityToken` 都需要从当前环境中获取
				- SDK 会从 `AWS_ROLE_ARN` 和 `AWS_WEB_IDENTITY_TOKEN_FILE` 环境变量中读取这些信息
				- 此外，SDK 还会尝试从 AWS 的配置文件中读取这些配置
		- ```http
		  https://sts.amazonaws.com/
		  ?Action=AssumeRoleWithWebIdentity
		  &DurationSeconds=3600
		  &PolicyArns.member.1.arn=arn:aws:iam::123456789012:policy/webidentitydemopolicy1
		  &PolicyArns.member.2.arn=arn:aws:iam::123456789012:policy/webidentitydemopolicy2
		  &ProviderId=www.amazon.com
		  &RoleSessionName=app1
		  &RoleArn=arn:aws:iam::123456789012:role/FederatedWebIdentityRole
		  &WebIdentityToken=Atza%7CIQEBLjAsAhRFiXuWpUXuRvQ9PZL3GMFcYevydwIUFAHZwXZXX
		  XXXXXXJnrulxKDHwy87oGKPznh0D6bEQZTSCzyoCtL_8S07pLpr0zMbn6w1lfVZKNTBdDansFB
		  mtGnIsIapjI6xKR02Yc_2bQ8LZbUXSGm6Ry6_BG7PrtLZtj_dfCTj92xNGed-CrKqjG7nPBjNI
		  L016GGvuS5gSvPRUxWES3VYfm1wl7WTI7jn-Pcb6M-buCgHhFOzTQxod27L9CqnOLio7N3gZAG
		  psp6n1-AJBOCJckcyXe2c6uD0srOJeZlKUm2eTDVMf8IehDVI0r1QOnTV6KzzAI3OY87Vd_cVMQ
		  &Version=2011-06-15
		  ```
	- Smaple Response
		- 注意这里的有效期，当有效期快到了的时候可以再次发起请求来换新 Token
		- ```xml
		  <AssumeRoleWithWebIdentityResponse xmlns="https://sts.amazonaws.com/doc/2011-06-15/">
		    <AssumeRoleWithWebIdentityResult>
		      <SubjectFromWebIdentityToken>amzn1.account.AF6RHO7KZU5XRVQJGXK6HB56KR2A</SubjectFromWebIdentityToken>
		      <Audience>client.5498841531868486423.1548@apps.example.com</Audience>
		      <AssumedRoleUser>
		        <Arn>arn:aws:sts::123456789012:assumed-role/FederatedWebIdentityRole/app1</Arn>
		        <AssumedRoleId>AROACLKWSDQRAOEXAMPLE:app1</AssumedRoleId>
		      </AssumedRoleUser>
		      <Credentials>
		        <SessionToken>AQoDYXdzEE0a8ANXXXXXXXXNO1ewxE5TijQyp+IEXAMPLE</SessionToken>
		        <SecretAccessKey>wJalrXUtnFEMI/K7MDENG/bPxRfiCYzEXAMPLEKEY</SecretAccessKey>
		        <Expiration>2014-10-24T23:00:23Z</Expiration>
		        <AccessKeyId>ASgeIAIOSFODNN7EXAMPLE</AccessKeyId>
		      </Credentials>
		      <SourceIdentity>SourceIdentityValue</SourceIdentity>
		      <Provider>www.amazon.com</Provider>
		    </AssumeRoleWithWebIdentityResult>
		    <ResponseMetadata>
		      <RequestId>ad4156e9-bce1-11e2-82e6-6b6efEXAMPLE</RequestId>
		    </ResponseMetadata>
		  </AssumeRoleWithWebIdentityResponse>
		  ```
	- 很多应用会在 [[AWS EKS]] 中注入这些变量
		- 创建一个 serviceaccount，然后在 pod 启动时指定环境变量
			- ```yaml
			  apiVersion: apps/v1
			  kind: Pod
			  metadata:
			    name: myapp
			  spec:
			    serviceAccountName: my-serviceaccount
			    containers:
			    - name: myapp
			      image: myapp:1.2
			      env:
			      - name: AWS_ROLE_ARN
			        value: arn:aws:iam::123456789012:role/eksctl-irptest-addon-iamsa-default-my-serviceaccount-Role1-UCGG6NDYZ3UE
			      - name: AWS_WEB_IDENTITY_TOKEN_FILE
			        value: /var/run/secrets/eks.amazonaws.com/serviceaccount/token
			      volumeMounts:
			      - mountPath: /var/run/secrets/eks.amazonaws.com/serviceaccount
			          name: aws-iam-token
			          readOnly: true
			    volumes:
			    - name: aws-iam-token
			      projected:
			        defaultMode: 420
			        sources:
			        - serviceAccountToken:
			            audience: sts.amazonaws.com
			            expirationSeconds: 86400
			            path: token
			  ```
			-
-
- 如何测试
	- AssumeRoleWithWebIdentity 看起来依赖一个 OIDC 来提供一个 token？
		- 可能要走一个 OAuth 的流程来拿 token？
			- [Using web identity federation API operations for mobile apps](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_providers_oidc_manual.html)
			- 最后选择了注册 [[Auth0]] - -
		- 我可以开一个服务？
			- <AssumeRoleWithWebIdentityResponse xmlns="https://sts.amazonaws.com/doc/2011-06-15/">
			    <AssumeRoleWithWebIdentityResult>
			      <Audience>test_audience</Audience>
			      <AssumedRoleUser>
			        <AssumedRoleId>role_id:reqsign</AssumedRoleId>
			        <Arn>arn:aws:sts::123:assumed-role/reqsign/reqsign</Arn>
			      </AssumedRoleUser>
			      <Provider>arn:aws:iam::123:oidc-provider/example.com/</Provider>
			      <Credentials>
			        <AccessKeyId>access_key_id</AccessKeyId>
			        <SecretAccessKey>sceret_access_ket</SecretAccessKey>
			        <SessionToken>session_token</SessionToken>
			        <Expiration>2022-05-25T11:45:17Z</Expiration>
			      </Credentials>
			      <SubjectFromWebIdentityToken>subject</SubjectFromWebIdentityToken>
			    </AssumeRoleWithWebIdentityResult>
			    <ResponseMetadata>
			      <RequestId>b1663ad1-23ab-45e9-b465-9af30b202eba</RequestId>
			    </ResponseMetadata>
			  </AssumeRoleWithWebIdentityResponse>
- 参考资料
	- [Introducing fine-grained IAM roles for service accounts](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/)