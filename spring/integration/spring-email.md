#  使用Spring email 发送邮件
{docsify-updated}

> https://docs.spring.io/spring-framework/reference/integration/email.html

CC：抄送，所有收件人都能看到
BCC：密送，只有发件人和服务器知道，其他人看不到

```
To:   alice@company.com
Bcc:  boss@company.com, audit@company.com
```

Alice 完全不知道 boss / audit 收到了邮件

邮件地址标准（RFC 5322）:
+ 多个收件人只能用 `,` 分隔
+ `;` 不是地址分隔符
+ `;` 在 RFC 里是 地址组（group）语法的一部分

## 配置与使用
```
spring.mail.host=smtp.gmail.com
spring.mail.protocol=smtps
spring.mail.port=465
spring.mail.username=perfect_a_day
spring.mail.password=dsaioi0njn9
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true

注入 JavaMailSender
@Resource
JavaMailSender javaMailSender;
```

更多配置属性可以参考 `MailProperties` 类

注意：默认超时时间被设置为无限，需要修改该值以避免线程被无响应的邮件服务器阻塞，如下例所示：
```
spring.mail.properties[mail.smtp.connectiontimeout]=5000
spring.mail.properties[mail.smtp.timeout]=3000
spring.mail.properties[mail.smtp.writetimeout]=5000
```

## 邮件支持HTML渲染
```
MimeMessage message = this.mailSender.createMimeMessage();
// Set From: header field of the header.
message.setFrom(new InternetAddress(from));
// Set To: header field of the header.
message.addRecipient(Message.RecipientType.TO, new InternetAddress(to));
// Set Subject: header field
message.setSubject("This is the Subject Line!");
// Now set the actual message
message.setText(body, "UTF-8", "html");
```

## MimeMessageHelper
```
// of course you would use DI in any real-world cases
JavaMailSenderImpl sender = new JavaMailSenderImpl();
sender.setHost("mail.host.com");

MimeMessage message = sender.createMimeMessage();
MimeMessageHelper helper = new MimeMessageHelper(message);
helper.setTo("test@host.com");
helper.setText("Thank you for ordering!");

sender.send(message);
```

## 发送附件
```
JavaMailSenderImpl sender = new JavaMailSenderImpl();
sender.setHost("mail.host.com");

MimeMessage message = sender.createMimeMessage();

// use the true flag to indicate you need a multipart message
MimeMessageHelper helper = new MimeMessageHelper(message, true);
helper.setTo("test@host.com");

helper.setText("Check out this image!");

// let's attach the infamous windows Sample file (this time copied to c:/)
FileSystemResource file = new FileSystemResource(new File("c:/Sample.jpg"));
helper.addAttachment("CoolImage.jpg", file);

sender.send(message);
```

## inline resource
```
MimeMessageHelper helper = new MimeMessageHelper(message, true);
helper.setTo("test@host.com");

// use the true flag to indicate the text included is HTML
helper.setText("<html><body><img src='cid:identifier1234'></body></html>", true);

// let's include the infamous windows Sample file (this time copied to c:/)
FileSystemResource res = new FileSystemResource(new File("c:/Sample.jpg"));
helper.addInline("identifier1234", res);
```

## 使用模板库创建电子邮件内容
然而在典型的企业应用中，开发人员通常不会采用前述方法创建邮件内容，原因如下：
+ 在Java代码中创建基于HTML的邮件内容既繁琐又易出错。
+ 显示逻辑与业务逻辑之间缺乏明确的分离。
+ 修改邮件内容的显示结构需要编写Java代码、重新编译、重新部署等操作。

通常解决这些问题的方案是采用模板库（如 `FreeMarker` ）来定义邮件内容的显示结构。这样代码只需负责生成邮件模板中待渲染的数据并发送邮件。当邮件内容达到中等复杂度时，这无疑是最佳实践方案。借助Spring框架对 `FreeMarker` 的支持类，实现起来相当便捷。


## 常见问题
1. org.springframework.mail.MailSendException: Failed messages: com.sun.mail.smtp.SMTPSendFailedException: 550 Invalid User
message.setFrom("perfect_a_day@163.com"); 填写正确的用户名

2. response: [EOF]. Failed messages: javax.mail.MessagingException: Got bad greeting from SMTP host: smtp.163.com, port: 465, response: [EOF]
升级协议为 smtps
```
spring.mail.host=smtp.gmail.com
spring.mail.protocol=smtps
spring.mail.port=465
spring.mail.username=perfect_a_day
spring.mail.password=dsaioi0njn9
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
```