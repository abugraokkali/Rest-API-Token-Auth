# Anahtar Doğrulamalı API Servisi Oluşturma

Her istekte kullanıcı adı ve parolayı gönderme zorunluluğu sakıncalıdır ve aktarım güvenli HTTP olsa bile bir güvenlik riski olarak görülebilir, çünkü istemcinin bu kimlik bilgilerini isteklerle gönderebilmesi için şifrelemeden saklaması gerekir.

Önceki çözüme göre bir gelişme, isteklerin kimliğini doğrulamak için bir anahtar kullanmaktır. Buradaki fikir, istemci uygulamasının bir kimlik doğrulama anahtarı için kimlik doğrulama bilgilerini değiş tokuş etmesi ve sonraki isteklerde bu anahtarı göndermesidir.

Anahtarlar genellikle bir sona erme süresi ile verilir, ardından geçersiz hale gelirler ve yeni bir anahtar alınması gerekir. Bir anahtarın sızdırılması durumunda oluşabilecek potansiyel hasar, kısa ömürleri nedeniyle çok daha küçüktür.
<br />
## Uygulama 
​	Anahtarları kullanmanın birçok yolu vardır:

Basit bir uygulama, veritabanında kullanıcı ve parola ile saklanan, muhtemelen bir son kullanma tarihi olan, belirli uzunlukta rastgele bir karakter dizisi oluşturmaktır.

Sunucu tarafında depolama gerektirmeyen daha ayrıntılı bir uygulama ise, kriptografik olarak imzalanmış bir mesajı anahtar olarak kullanmaktır. Bunun avantajı, anahtarla ilgili bilgilerin, yani anahtarın üretildiği kullanıcının, anahtarın kendisinde kodlanması ve güçlü bir kriptografik imza ile korunmasıdır.

Bu uygulamada da benzer bir yaklaşım kullanacağız.
<br />
## Kod

Başlangıçta Python ile basit bir REST-API oluşturmak için [burayı](https://github.com/abugraokkali/Rest-API) inceleyebilirsiniz. 

```python
from flask import Flask, jsonify, request, make_response
import jwt 
import datetime
from functools import wraps
```

- Gerekli paketlerin import edilmesi

  **jwt**: JSON Web Token'lerini (JWT) kodlamanıza ve kodunu çözmenize olanak tanır.

  **datetime**: Datetime modülü, tarih ve saatle çalışmak için sınıflar sağlar.

  **functools**, daha yüksek dereceli fonksiyonlar (diğer fonksiyonlar üzerinde hareket eden veya başka fonksiyon döndüren fonksiyonlar) için standart bir Python modülüdür. **wraps()**, bir dekoratörün sarmalayıcı işlevine uygulanan bir dekoratördür.

```python
app = Flask(__name__)

app.config['SECRET_KEY'] = 'thisisthesecretkey'
```

- Flask app objesinin ve gizli anahtarın oluşturulması.

```python
def token_required(f):
    @wraps(f)
    def decorated(*args, **kwargs):
        token = request.args.get('token')

        if not token:
            return jsonify({'message' : 'Token is missing!'}), 403

        try: 
            data = jwt.decode(token, app.config['SECRET_KEY'], algorithms="HS256")
        except Exception as inst:
            print(inst)
            return jsonify({'message' : 'Token is invalid!'}), 403

        return f(*args, **kwargs)

    return decorated
```

- Anahtar girilmediyse veya girilen anahtar hatalıysa hata mesajları basan, girilen doğru bir şekilde decode edildiyse devam eden wrapper fonksiyon.

```python
@app.route('/unprotected')
def unprotected():
    return jsonify({'message' : 'Anyone can view this!'})

@app.route('/protected')
@token_required
def protected():
    return jsonify({'message' : 'This is only available for people with valid tokens.'})

```

- Test aşamasında kullanmak için anahtar gerektiren ve gerektirmeyen end-point'lerin oluşturulması.

```python
@app.route('/login')
def login():
    auth = request.authorization

    if auth and auth.password == 'Passw0rd':
        token = jwt.encode({'user' : auth.username, 'exp' : datetime.datetime.utcnow() + datetime.timedelta(minutes=15)}, app.config['SECRET_KEY'], algorithm="HS256")

        return jsonify({'token' : token})

    return make_response('Could not verify!', 401, {'WWW-Authenticate' : 'Basic realm="Login Required"'})
```

- Herhangi bir kullanıcı adı ve "PasswOrd"parolası ile /login end-point'inini çağırdığınızda ;
  - kullanıcı adının
  - son kullanma tarihinin
  - ve gizli anahtarın 

jwt ile oluşturulan anahtarı döndüren fonksiyon. datetime'ı kullanıp anahtarın kullanılabilirlik süresini 15 dakika yaptığımızı da görebilirsiniz.

<br />

##Çalıştırma

```bash
$ python3 api.py

 * Serving Flask app "api" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
 * Running on http://127.0.0.1:5000/ (Press CTRL+C to quit)
 * Restarting with stat
 * Debugger is active!
 * Debugger PIN: 241-307-717
```
<br />

## Testler

Testleri tarayıcınız üzerinde verilen adreslere giderek yapabilirsiniz.

```
http://127.0.0.1:5000/unprotected
```

adresine gittiğimizde beklediğimiz içeriği görebiliyoruz.

![ss1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1o858oz2yc2cpmg8n0x5.png)

```
http://127.0.0.1:5000/protected
```

adresine gittiğimizde bir anahtar beklendiği için içerikte anahtar eksik uyarısı alıyoruz.

![ss2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/dgvr7u3jytpksn06ob1o.png)

```
http://127.0.0.1:5000/login
```

Adresine gittiğimizde bizden kullanıcı adı ve şifre isteniyor. Herhangi bir kullanıcı adı ve "Passw0rd" şifresiyle oturum açalım.

![ss3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/khk2wuapjirqxz2wmvsi.png)

Oturum açtığımızda kullanıcı adımıza ve gizli anahtarımıza özel oluşturulan 15 dakika geçerli anahtarımızı görüntüleyebiliyoruz.

![ss4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/ev3q80l8q6deq7r4yqoh.png)

```
http://127.0.0.1:5000/protected?token=invalidtoken
```

Anahtar gerektiren adrese yanlış bir anahtar ile gitmeye çalıştığımızda beklediğimiz uyarıyla karşılaşıyoruz.

![ss5](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/487mpc16lr1g6x3b9cqm.png)

```
http://127.0.0.1:5000/protected?token=eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyIjoiYWxpIiwiZXhwIjoxNjI5MTg4NDMwfQ.ni3Soivc1a4vKyI3_xpDyb1-RV3iDQ4QMtS3FhXijog
```

Aynı adrese bu şekilde gittiğimizde ise herhangi bir uyarıyla karşılaşmadan içeriği görebiliyoruz.

![ss6](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9kqqxtq5z61sblemzdtx.png)

Bu repository'nin dev.to yazısına [linkten](https://dev.to/aciklab/anahtar-dogrulamali-api-servisi-olusturma-5gbo) ulaşabilirsiniz.
