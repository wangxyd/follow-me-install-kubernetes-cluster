# A.浏览器访问 kube-apiserver 安全端口

浏览器访问 kube-apiserver 的安全端口 8443 时，提示证书不被信任：
![DLG_FLAGS_INVALID_CA](images/invalid-ca.png)

这是因为 kube-apiserver 的 server 证书是我们创建的根证书 ca.pem 签名的，需要将根证书 ca.pem 导入操作系统，并设置永久信任。

对于 Mac，操作如下：

![keychain-mac](images/keychain-mac.png)

对于 Windows，操作如下：Internet属性-->内容-->证书-->导入-->浏览选择“ca.pem”-->选择“将所有的证书都放入下列存储”，证书存储选择“受信任的根证书颁发机构”-->完成。

![keychain-win](images/keychain-win.png)

或者使用 Java 证书工具 keytool 导入ca.perm
``` bash
keytool -import -v -trustcacerts -alias appmanagement -file "PATH...\\ca.pem" -storepass password -keystore cacerts
```

再次访问 https://192.168.99.100:8443/，已信任，但提示 401，未授权的访问：

![browser-unauthorized](images/browser-unauthorized.png)

我们需要给浏览器生成一个 client 证书，访问 apiserver 的 8443 https 端口时使用。

这里使用部署 kubectl 命令行工具时创建的 admin 证书、私钥和上面的 ca 证书，创建一个浏览器可以使用 PKCS#12/PFX 格式的证书：

``` bash
[admin@k8s-admin ~]$ openssl pkcs12 -export -out cert/kubectl.pfx -inkey cert/kubectl-key.pem -in cert/kubectl.pem -certfile cert/ca.pem
Enter Export Password:
Verifying - Enter Export Password:
[admin@k8s-admin ~]$ ls cert/kubectl.pfx
cert/kubectl.pfx
```

将创建的 kubectl.pfx 导入到系统的证书中。

对于 Mac，操作如下：
![clent-cert-mac](images/client-cert-mac.png)

对于 Windows，操作如下：
![client-cert-win](images/client-cert-win.png)

**重启浏览器**，再次访问 https://192.168.99.100:8443/，提示选择一个浏览器证书，这里选中上面导入的 kubectl.pfx：

![client-cert-select](images/client-cert-select.png)

这一次，被授权访问 kube-apiserver 的安全端口：

![browser-authorized](images/browser-authorized.png)

## 客户端选择证书的原理

1. 证书选择是在客户端和服务端 SSL/TLS 握手协商阶段商定的；
1. 服务端如果要求客户端提供证书，则在握手时会向客户端发送一个它接受的 CA 列表；
1. 客户端查找它的证书列表(一般是操作系统的证书，对于 Mac 为 keychain)，看有没有被 CA 签名的证书，如果有，则将它们提供给用户选择（证书的私钥）；
1. 用户选择一个证书私钥，然后客户端将使用它和服务端通信。


## 参考
+ https://github.com/kubernetes/kubernetes/issues/31665
+ https://www.sslshopper.com/ssl-converter.html
+ https://stackoverflow.com/questions/40847638/how-chrome-browser-know-which-client-certificate-to-prompt-for-a-site
