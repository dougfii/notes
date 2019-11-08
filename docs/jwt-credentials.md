# JWT Token Credentials

+ 通过keytool生成证书
```
# keytool -genkeypair -alias gomro -keyalg RSA -keypass echoshen -storepass echoshen -keystore gomro-galaxy.jks
```
+ 查看证书信息
```
# keytool -list -v -storepass echoshen -keystore gomro_galaxy.jks
```
+ 查看公钥信息
```
# keytool -list -rfc -storepass echoshen -keystore gomro_galaxy.jks
```