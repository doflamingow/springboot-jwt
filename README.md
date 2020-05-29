# Belajar Spring Security dengan OAuth 2 #

Fitur aplikasi:

* Grant type user password
* Grant type implicit

## Struktur Aplikasi ##

Contoh kode program ini melibatkan beberapa pihak, yaitu:

* `auth-server` : authorization server, memegang username dan password dan bertugas melakukan otentikasi
* `resource-server` : penyedia layanan (API) yang ingin diakses dari aplikasi. Resource server tidak melakukan pemeriksaan username/password, dia mendelegasikan otentikasi ke `auth-server`
* `client-application` : aplikasi yang dihadapi user. Aplikasi ini menggunakan layanan dari `resource-server`
* `resource-owner` : pemilik data dalam `resource-server`. Disebut juga dengan istilah `user` pada `client-application`.

Pada prakteknya, ada beberapa skenario deployment yang memungkinkan:

1. Authorization dan Resource Server di satu aplikasi. Untuk skenario ini, agar bisa berkoordinasi, kita bisa menggunakan InMemoryTokenStore yang digunakan bersama oleh authorization dan resource server
2. Authorization Server pisah aplikasi (beda war) dengan Resource Server. Ada dua pilihan metode koordinasi token, yaitu:

    * Remote Token Service: Authorization server menyimpan token di InMemoryTokenStore, sedangkan Resource server menggunakan akses http ke authorization server untuk mengecek validitas token. Contohnya ada [di sini](./resource-server/src/main/java/com/muhardin/endy/belajar/springoauth2/resourceserver/Oauth2ResourceServer.java#L37).
    * Sharing Database: Authorization server menyimpan token di database, dan Resource server mengakses database yang sama untuk mengecek validitas token. Skema database default yang digunakan Spring OAuth bisa dilihat [di sini](https://github.com/spring-projects/spring-security-oauth/blob/master/spring-security-oauth2/src/test/resources/schema.sql).

## Menjalankan Aplikasi ##

Aplikasi `authorization-server`, `resource-server`, maupun `client` dijalankan masing-masing secara terpisah. Walaupun demikian, semua aplikasi ini menggunakan username/password yang sama.

User yang tersedia :

| Username | Password | Role  |
|:--------:|:--------:|:-----:|
| endy     | 123      | admin |
| anton    | 123      | staff |

Karena ini adalah aplikasi OAuth, maka kita perlu mendaftarkan beberapa aplikasi client ke `authorization-server`. Berikut adalah daftar client, jenis akses (grant type), dan passwordnya


| Client         | Grant Type         | Client Secret  |
|:--------------:|:------------------:|:--------------:|
| clientauthcode | authorization_code | 123456         |
| clientapp      | password           | 123456         |
| jsclient       | implicit           | jspasswd       |


### Authorization Server ###

Aplikasi `auth-server` menyediakan beberapa layanan :

* login : `http://localhost:10000/login.jsp`
* request token : Ada dua url yang digunakan untuk request token, tergantung grant type yang kita gunakan, apakah `user password` atau `implicit`.
* Untuk grant type `user password` atau dikenal juga dengan istilah `resource owner credential`, gunakan url `http://localhost:10000/oauth/token`
* Untuk grant type `implicit`, gunakan url `http://localhost:10000/oauth/authorize`
* Setelah token didapatkan, `resource-server` biasanya tidak percaya begitu saja. Untuk itu `auth-server` menyediakan url `http://localhost:10000/oauth/check_token` untuk mengecek validitas token.

Detail konfigurasi pemanggilan masing-masing URL bisa dilihat pada contoh kasus di bawah.

Aplikasi `auth-server` bisa dijalankan dengan cara sebagai berikut:

1. Masuk ke folder `authorization-server`

        cd authorization-server

2. Jalankan aplikasinya

        mvn clean spring-boot:run

3. Siap digunakan di `http://localhost:10000`

### Resource Server ###

Aplikasi `resource-server` menyediakan 3 layanan : 

* `http://localhost:10001/api/halo` : boleh diakses tanpa otentikasi
* `http://localhost:10001/api/admin` : hanya boleh diakses role admin
* `http://localhost:10001/api/staff` : hanya boleh diakses role staff

Aplikasi `resource-server` bisa dijalankan dengan cara sebagai berikut:

1. Masuk ke folder `authorization-server`

        cd resource-server

2. Jalankan aplikasinya

        mvn clean spring-boot:run

3. Siap digunakan di `http://localhost:10001`

### Client Apps ###

Aplikasi `implicit-client` bisa dijalankan dengan cara sebagai berikut:

1. Masuk ke folder `implicit-client`

        cd implicit-client

2. Jalankan aplikasinya

        mvn clean spring-boot:run

3. Siap digunakan di `http://localhost:20000/implicit-client/`

## Contoh Kasus ##

OAuth memiliki beberapa grant type, yaitu:

* Authorization Code
* Implicit
* User Password
* Client Credentials

Flow `authorization code` digunakan bila kita ingin memverifikasi identitas aplikasi client. Ini biasa digunakan apabila aplikasi clientnya bisa menyimpan `client secret` dengan aman, misalnya pada aplikasi web server side (PHP, Java, NodeJS, Python, Ruby, dsb). Dalam flow ini, setelah user login, dia akan mendapatkan `authorization code`. `Authorization code` ini diserahkan kepada aplikasi client, untuk kemudian ditukarkan dengan `access token`. Untuk selanjutnya, aplikasi client mengakses `resource server` dengan membawa `access token`. Pada flow ini, `access token` sama sekali tidak dikirim ke user, sehingga meminimasi potensi kebocoran antara user dan aplikasi client.

Flow `implicit` digunakan bila kita tidak bisa atau tidak perlu memverifikasi aplikasi client. Ini biasanya terjadi bila aplikasi clientnya bisa diunduh dan diakses secara bebas, misalnya aplikasi berbasis JavaScript client side (jQuery, AngularJS, EmberJS, Vue.js, dsb) ataupun aplikasi mobile (Android, iOS). Karena aplikasi clientnya bertebaran tanpa kontrol, maka tidak perlu diverifikasi, tidak ada manfaatnya. Dengan demikian, dalam flow ini setelah login user langsung diberikan `access token` untuk selanjutnya digunakan untuk mengakses resource server. Resiko keamanannya lebih tinggi, karena `access token` dibawa setiap kali melakukan request, dimana koneksi antara user dan resource server lebih sulit diamankan (misalnya di public wifi).

Flow `user password` digunakan apabila aplikasi client 100% terpercaya, misalnya aplikasi mobile yang dibuat sendiri. Dengan demikian, kita bisa mengijinkan aplikasi client untuk langsung meminta username dan password user.

Flow `client credential` sama sekali tidak melibatkan user. Ini digunakan untuk mengamankan koneksi dari aplikasi ke aplikasi. Misalnya aplikasi client ingin mengakses fitur umum dari resource server yang tidak berkaitan dengan user tertentu (misalnya jumlah semua user). Konsekuensinya, dengan flow ini kita tidak bisa membatasi akses spesifik ke satu user tertentu.

### Flow Grant Type Authorization Code ###

Grant type ini digunakan untuk aplikasi client yang bisa menyimpan nilai `client secret`. Contohnya adalah aplikasi server side (PHP, Java) atau aplikasi desktop/mobile yang bisa dicompile. Nilai `client secret` bisa kita simpan sebagai variabel yang tidak bisa dilihat umum.

* Jalankan aplikasi `authorization-server` dan `resource-server`
* Buka browser, arahkan ke 

        http://localhost:10000/oauth/authorize?client_id=clientauthcode&response_type=code&redirect_uri=http://localhost:10001/api/state/new

* Kita akan diminta login dan kemudian melakukan otorisasi. Setelah login, kita akan disajikan halaman otorisasi, pilih radio button Approve kemudian klik tombol `Authorize`.

* Kita akan diredirect ke url yang kita sebutkan pada variabel `redirect_uri` di langkah kedua di atas dengan ditambahi parameter authorization code. URL hasil redirectnya seperti ini:

        http://localhost:10001/api/state/new?code=8OppiR

    Ambil kodenya, yaitu `8OppiR`, untuk digunakan pada langkah selanjutnya.

* Lakukan request dari aplikasi client untuk menukar authorization code dengan access token. Sebagai contoh, kita akan gunakan aplikasi commandline `curl`. Berikut perintahnya

        curl -X POST -vu clientauthcode:123456 http://localhost:10000/oauth/token -H "Accept: application/json" -d "grant_type=authorization_code&code=iMAtdP&redirect_uri=http://localhost:10001/api/state/new"

* Kita akan diberikan access token dalam response JSON seperti ini

        {
            "access_token":"08664d93-41e3-473c-b5d2-f2b30afe7053",
            "token_type":"bearer",
            "refresh_token":"436761f1-2f26-412b-ab0f-bbf2cd7459c4",
            "expires_in":43199,
            "scope":"write read"
        }

* Access token tersebut bisa digunakan aplikasi client untuk mengakses resource terproteksi seperti ini

        curl http://localhost:10001/api/admin?access_token=08664d93-41e3-473c-b5d2-f2b30afe7053

* Resource server mengecek ke authorization server apakah token tersebut valid atau tidak dengan HTTP request seperti ini

        curl -X POST -vu clientauthcode:123456 http://localhost:10000/oauth/check_token?token=08664d93-41e3-473c-b5d2-f2b30afe7053

* Authorization server akan membalas request check token dengan response seperti ini


        {
            "aud": ["belajar"],
            "exp": 1444158090,
            "user_name": "endy",
            "authorities": ["ROLE_OPERATOR", "ROLE_SUPERVISOR"],
            "client_id": "clientauthcode",
            "scope": ["read", "write"]
        }


* Resource server akan memberikan data yang diminta karena tokennya cocok

        {"sukses":true,"page":"admin","user":"endy"}

* Bila access token expire, kita bisa meminta refresh token sebagai berikut

        curl -X POST -vu clientauthcode:123456 http://localhost:10000/oauth/token -d "client_id=clientauthcode&grant_type=refresh_token&refresh_token=436761f1-2f26-412b-ab0f-bbf2cd7459c4"

* Authorization server akan memberikan respon access token dan refresh token yang baru

        {
            "access_token":"e425cee6-7167-4eea-91c3-2706d01dab7f",
            "token_type":"bearer",
            "refresh_token":"436761f1-2f26-412b-ab0f-bbf2cd7459c4",
            "expires_in":43199,"scope":"write read"
        }

* Flow tersebut bisa digambarkan sebagai berikut

![Flow Auth Code](./img/flow-oauth2-authcode.jpg)

### Flow Grant Type User Password ##

Grant type ini biasanya digunakan bila pembuat aplikasi client sama dengan pembuat resource server. Sehingga aplikasi client diperbolehkan mengambil data username dan password langsung dari user. Contohnya: aplikasi Twitter android ingin mengakses daftar tweet untuk user tertentu. Walaupun demikian, penggunaan flow type ini tidak direkomendasikan lagi. Sebaiknya gunakan flow type _authorization code_ atau _client credentials_.

* Jalankan Aplikasi `authorization-server` dan `resource-server`
* Akses url terproteksi tanpa login dulu

        curl http://localhost:10001/api/admin 
    
    Ini akan menghasilkan response code `401`

* Request token dulu : 

        curl -X POST -vu clientapp:123456 http://localhost:10000/oauth/token -H "Accept: application/json" -d "client_id=clientapp&grant_type=password&username=endy&password=123" 

    Ini akan menghasilkan response JSON berisi token, misalnya tokennya adalah `initokensaya`

* Akses lagi url terproteksi dengan menggunakan token : 

        curl -H "Authorization: Bearer initokensaya" http://localhost:10001/api/admin

    Ini akan menghasilkan response sukses.


### Flow Grant Type Implicit ###

Grant type ini biasanya digunakan apabila aplikasi client tidak bisa menyimpan nilai `client secret` dengan aman. Contohnya adalah aplikasi JavaScript (misalnya: AngularJS, jQuery, Backbone.js, dsb) yang source codenya bisa dilihat umum.

* Jalankan aplikasi `authorization-server` dan `resource-server`

* Generate random variabel `state` dulu supaya aman

        curl http://localhost:10001/api/state/new
        
    Misalnya, nilai variabel `state` yang didapatkan `blablabla`. Variabel `state` ini disimpan sebagai session variable di sisi server pada aplikasi client. Kita akan gunakan nilai ini untuk verifikasi pada langkah selanjutnya.

* Setelah dapat variabel `state` dari request di atas, gunakan untuk generate token

        curl http://localhost:10000/oauth/authorize?client_id=jsclient&response_type=token&scope=write&state=blablabla

* Bila kita belum login, authorization server akan redirect kita ke halaman login. Silahkan isi username/password sesuai tabel di atas.

* Setelah sukses login, authorization server akan melakukan redirect ke url yang kita daftarkan, yaitu `http://localhost:10001/api/state/verify`. URL ini akan ditambahkan hash variable berisi token, sehingga isi lengkapnya seperti ini
    
        http://localhost:10001/api/state/verify#access_token=667aadee-883c-439f-9f18-50ef77e3fad6&token_type=bearer&state=blablabla&expires_in=43199

* Di aplikasi client kita, URL redirect ini kita program supaya mengembalikan nilai variabel `state` yang sudah kita set pada langkah pertama. Selanjutnya, kita bandingkan nilai state yang dikembalikan server kita dengan nilai state yang dikembalikan authorization server. Karena isinya sama-sama `blablabla`, kita yakin bahwa `access_token` yang dihasilkan memang benar-benar valid.

* Sekarang kita bisa mengakses URL yang kita inginkan

        curl http://localhost:10001/api/admin?access_token=667aadee-883c-439f-9f18-50ef77e3fad6

* Selain ditaruh di request parameter, kita juga bisa memasang access token di header `Authorization` seperti ini:

        curl -H "Authorization: Bearer 667aadee-883c-439f-9f18-50ef77e3fad6" http://localhost:10001/api/admin
    
    Ini memudahkan kita untuk memasang `$httpInterceptor` bila kita menggunakan AngularJS.

* Di belakang layar, resource server akan melakukan menanyakan validitas token ke authorization server dengan perintah seperti ini

        curl -X POST -vu jsclient:jspasswd http://localhost:10000/oauth/check_token?token=667aadee-883c-439f-9f18-50ef77e3fad6
    
    Bila token valid, authorization server akan menjawab dengan detail informasi pemegang token, seperti ini
    
        {
            "exp": 1401385717,
            "user_name": "endy",
            "scope": ["write"],
            "authorities": ["ROLE_ADMIN"],
            "aud": ["belajar"],
            "client_id": "jsclient"
        }


### Flow Grant Type Client Credentials ###

Pada flow type ini, aplikasi client diberikan akses penuh terhadap resource yang diproteksi tanpa perlu meminta username dan password user. Biasanya digunakan bila aplikasi client dan aplikasi resource server dibuat oleh perusahaan yang sama.

* Jalankan aplikasi `authorization-server` dan `resource-server`

* Request token ke `authorization-server` dengan memasang `client_id` dan `client_secret` pada header dengan cara Basic Authentication

        curl -X POST -vu clientcred:123456 http://localhost:10000/oauth/token -H "Accept: application/json" -d "grant_type=client_credentials"

* Kita akan mendapatkan response berupa `access_token` dalam format JSON

        {
            "access_token":"45841c94-8851-4f93-bdb1-7de9519df175",
            "token_type":"bearer",
            "expires_in":43199,"scope":"trust"
        }

* Gunakan access token ini dalam Authorization header untuk mengakses `resource-server`

        curl -H "Authorization: Bearer 45841c94-8851-4f93-bdb1-7de9519df175" http://localhost:10001/api/client

* Sukses menghasilkan response

        {
            "sukses":true,
            "page":"client",
            "user":"clientcred"
        }

## Referensi ##

Konsep OAuth

* https://www.digitalocean.com/community/tutorials/an-introduction-to-oauth-2
* http://tutorials.jenkov.com/oauth2/index.html
* https://aaronparecki.com/2012/07/29/2/oauth2-simplified
* http://www.bubblecode.net/en/2016/01/22/understanding-oauth2/
* http://stackoverflow.com/questions/16321455/what-is-the-difference-between-the-2-workflows-when-to-use-authorization-code-f
* http://oauth2.thephpleague.com/authorization-server/which-grant/

Server Side

* https://spring.io/guides/tutorials/spring-security-and-angular-js/
* http://tools.ietf.org/id/draft-ietf-oauth-v2-31.html
* http://techblog.hybris.com/2012/06/01/oauth2-authorization-code-flow/
* http://techblog.hybris.com/2012/06/05/oauth2-refreshing-an-expired-access-token/
* http://techblog.hybris.com/2012/06/05/oauth2-the-implicit-flow-aka-as-the-client-side-flow/
* http://techblog.hybris.com/2012/06/11/oauth2-resource-owner-password-flow/
* http://techblog.hybris.com/2012/07/09/oauth-the-client-credentials-flow/
* http://techblog.hybris.com/2012/08/10/oauth2-server-side-implementation-using-spring-security-oauth2-module/
* http://projects.spring.io/spring-security-oauth/docs/oauth2.html
* https://github.com/royclarkson/spring-rest-service-oauth
* http://spring.io/blog/2011/11/30/cross-site-request-forgery-and-oauth2
* https://github.com/spring-projects/spring-security-oauth/
* https://github.com/spring-projects/spring-security-oauth/blob/master/spring-security-oauth2/src/test/resources/schema.sql

Client Side

* http://www.frederiknakstad.com/2014/02/09/ui-router-in-angular-client-side-auth/
* http://nils-blum-oeste.net/cors-api-with-oauth2-authentication-using-rails-and-angularjs/
* https://auth0.com/blog/2014/01/07/angularjs-authentication-with-cookies-vs-token/
* https://auth0.com/blog/2014/01/27/ten-things-you-should-know-about-tokens-and-cookies/
* https://developers.google.com/accounts/docs/OAuth2UserAgent
* http://beletsky.net/2013/11/simple-authentication-in-angular-dot-js-app.html
* http://techblog.constantcontact.com/software-development/implementing-oauth2-0-in-an-android-app/
