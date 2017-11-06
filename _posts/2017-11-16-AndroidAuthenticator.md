---
layout: post
title: On Android Authenticator
---

++ Add account

https://developer.android.com/training/id-auth/custom_auth.html#AccountCode

+++ The first thing you'll need is a way to get credentials from the user.
  # Collects credentials from the user 获取用户证书，比如用户名、密码
  # Authenticates the credentials with the server 与服务器的交互及验证
  # Stores the credentials on the device  在设备上存储证书， 即创建账号.
the Android framework supplies a base class, AccountAuthenticatorActivity, which you can extend to create your own custom authenticator.
通常可以通过继承AccountAuthenticatorActivity定制authenticator.

    The third requirement has a canonical, and rather simple, implementation:
第三步的典型代码范式如下
final Account account = new Account(mUsername, your_account_type);
mAccountManager.addAccountExplicitly(account, mPassword, null);

+++ Be Smart About Security!
 * AccountManager 用明文保存密码，所以需要考虑安全问题. 通常用Auth-token的方式解决.
 
+++ Extend AbstractAccountAuthenticator
In order for the AccountManager to work with your custom account code, you need a class that implements the interfaces that AccountManager expects. This class is the authenticator class.

The easiest way to create an authenticator class is to extend AbstractAccountAuthenticator and implement its abstract methods.

You can find a step-by-step guide to implementing a successful authenticator class and the XML files in the AbstractAccountAuthenticator documentation. There's also a sample implementation in the SampleSyncAdapter sample app.

+++ Create an Authenticator Service 作用？？


+++ account manager
Pros: Standard way to authenticate users. Simplifies the process for the developer. Handles access-denied scenarios for you. Can handle multiple token types for a single account (e.g. Read-only vs. Full-access). Easily shares auth-tokens between apps. Great support for background processes such as SyncAdapters. Plus, you get a cool entry for your account type on the phone’s settings.

++++ Auth-token
Authentication Token (auth-token) -  A temporary access token (or security-token) given by the server. The user needs to identify to get such token and attach it to every request he sends to the server.

++++ AccountAuthenticator 
- A module to handle a specific account type. The AccountManager find the appropriate AccountAuthenticator talks with it to perform all the actions on the account type. The AccountAuthenticator knows which activity to show the user for entering his credentials and where to find any stored auth-token that the server has returned previously. 

++++ AccountAuthenticatorActivity 
- Base class for the “sign-in/create account” activity to be called by the authenticator when the user needs to identify himself. The activity is in charge of the sign-in or account creation process against the server and return an auth-token back to the calling authenticator.

++++ App登录过程
  # app首次登录， 调用AccountManager获取 authToken.
  # AccountManager查询 AccountAuthenticator是否可以获得authToken
  # 因为首次登录还没有获取过authToken, 所以启动AccountAuthenticatorActivity, 提示用户登录账号
  # 用户登录， 获得server返回的 authToken
  # AccountManager存储authToken供将来使用
  # app得到了authToken

+++ Creating our Authenticator
  * addAccount 
Called when the user wants to log-in and add a new account to the device.
可以从app本身或settings app调用 AccountManager#addAccount()

+++ getAuthToken

+++ Service
将Authenticator信息暴露给系统
All we want to do, is letting other processes bind with our service and communicate with our Authenticator.


