# Selfhosted git-gateway and gotrue

I used debian 9 stretch for this tutorial.

## Dependencies

#### During deployment:
- mariadb v10.2+ or mysql v5.7+ (for gotrue)
- sqlite3 (for git-gateway)

#### For building:
- golang-go v1.10+
- make

## MariaDB

### Installation

Debian 9 Stretch comes with mariadb 10.1, but gotrue needs at least mariadb v10.2.
Installing 10.3 by following the official instructions from [mariadb.org](https://downloads.mariadb.org/mariadb/repositories/#mirror=klaus&distro=Debian&distro_release=stretch--stretch&version=10.3):


Install the requirements, add keys from keyserver, add repository:

```
# apt-get install software-properties-common dirmngr
# apt-key adv --recv-keys --keyserver keyserver.ubuntu.com 0xF1656F24C74CD1D8
# add-apt-repository 'deb [arch=amd64,i386,ppc64el] http://mirror.klaus-uwe.me/mariadb/repo/10.3/debian stretch main'
```

Before upgrading mariadb be sure to make a backup of your existing databases first!
Run system update and install mariadb:

```
# apt-get update
# apt-get install mariadb-server
```

### Setup Database

Login to mariadb as the root user and create database and database-user for gotrue:

```
$ mysql -u root -p

mysql> CREATE DATABASE gotrue;
mysql> CREATE USER 'gotrue'@'localhost' IDENTIFIED BY 'password';
mysql> GRANT ALL PRIVILEGES ON gotrue.* TO gotrue@localhost;
mysql> FLUSH PRIVILEGES;
mysql> quit
```

## Golang Go

### Installation
For compiling gotrue as well as git-gateway we need at least go version 1.10+. Go has been developing fastly in the last years.
So the debian binary is way too old. The newest stable version can be installed by following their [official installation instructions](https://golang.org/doc/install) on
golang.org .


### Setup go environment
to permanently setup go's paths add the following lines to your .bashrc . For this to take effect you will need to logout and login again.

```
export GOPATH=$HOME/go
export PATH=$PATH:$GOROOT/bin:$GOPATH/bin
export PATH=$PATH:/usr/local/go/bin
```

## gotrue

#### Get the source

```
mkdir -p $GOPATH/src/github.com/netlify
cd $GOPATH/src/github.com/netlify
git clone https://github.com/netlify/gotrue
cd gotrue
```

#### Patch the source
Add the patch as described [here](https://github.com/netlify/gotrue/pull/188#issuecomment-435651982) (may be already fixed)


#### Build
In order automatically download the dependencies for gotrue listed in the mod.go file we have to enable the following go module (the digits should represent the go version you use):

```
export GO111MODULE=on 
```
Finally fetch dependencies and build gotrue:

```
make deps
make build
```

### Configuration

gotrue takes configuration via environment variables. One solution is to put them in a file called .env which has to be in the same folder as the gotrue binary and add the GOTRUE Prefix to each variable-name, like the example.env shows. I should also be possible to execute gotrue with the --config config.cfg flag, but I haven't tested that.

```
GOTRUE_JWT_SECRET="secret-key-shared-between-git-gateway-and-gotrue"
GOTRUE_JWT_EXP=3600
GOTRUE_JWT_AUD=""
GOTRUE_DB_DRIVER=mysql
DATABASE_URL="gotrue:password@tcp(127.0.0.1:3306)/gotrue?parseTime=true&multiStatements=true"
GOTRUE_API_HOST=localhost
PORT=8081
GOTRUE_SITE_URL=https://example.com
GOTRUE_LOG_LEVEL=DEBUG
GOTRUE_OPERATOR_TOKEN=super-secret-operator-token
GOTRUE_DISABLE_SIGNUP=false
```

If you have DISABLE_SIGNUP=false you probably want add the following part to your config. I skipped this. For my usecase it's enough to create accounts locally and to disable signup completely afterwards.

```
GOTRUE_SMTP_HOST=
GOTRUE_SMTP_PORT=
GOTRUE_SMTP_USER=
GOTRUE_SMTP_PASS=
GOTRUE_SMTP_ADMIN_EMAIL=
GOTRUE_MAILER_SUBJECTS_CONFIRMATION="Welcome to GoTrue!"
GOTRUE_MAILER_SUBJECTS_RECOVERY="Reset your GoTrue password!"
```

Now we have to initialize the gotrue database in order to get the basic API-functionality working. Migrate the database structure 

```
gotrue migrate
```

Not that the this commands reads the migrations from the files stored in the folder ./migrations.


## git-gateway
### Install dependencies

```
# apt-get install sqlite3
```

### Get source and build it

The procedure with git-gateway is quite similar to gotrue:

```
cd $GOPATH/src/github.com/netlify
git clone https://github.com/netlify/git-gateway.git
cd git-gateway

make deps
make build
```

### Configuration

git-gateway's configuration is also similar to gotrue's. Create an .env file like this:
Be shure to use the same JWT_SECRET as in gotrue. This key is used to verify JWT-Tokens (thats the way git-gateway and gotrue talk to one-another).

```
GITGATEWAY_JWT_SECRET="secret-key-shared-between-git-gateway-and-gotrue"

GITGATEWAY_DB_DRIVER=sqlite3
DATABASE_URL=gorm.db

GITGATEWAY_API_HOST=example.com
PORT=9999

GITGATEWAY_GITLAB_ACCESS_TOKEN="token-generated-on-the-gitlab-interface"
GITGATEWAY_GITLAB_ACCESS_TOKEN_TYPE="personal_access"
GITGATEWAY_GITLAB_REPO="owner/repo"

GITGATEWAY_ROLES="admin,editor"
```

If you want to use another GitLab Instance, you also have to set.

```
GITGATEWAY_GITLAB_ENDPOINT="https://custom-gitlab-instance.domain/api/v4"
```

Setting GITLAB_ACCESS_TOKEN_TYPE="oauth" is also possible.

To use GitHub delete the Gitlab enviroment variables and add the corresponding GitLab ones. Note that ACCESS_TOKEN_TYPE doesn't exist for GitHub.

```
GITGATEWAY_GITHUB_ACCESS_TOKEN="token-generated-on-the-gitlab-interface"
GITGATEWAY_GITHUB_REPO="owner/repo"
```

## Running

You can just execute git-gateway and gotrue without further flags. Feel free to run them as a service and share your service files (systemd) here.

## How to manage GoTrue Users

Now we have the services running. In order for gotrue to be useful, we need to create our first user, or even better an admin user.

You would expect the following to work properly:

```
./gotrue admin createuser example@domain.org password
```

This is not working because the Instance which is currently running has the Instance-ID 0 and the generation of the corresponding UUID fails. Note the --superadmin flag. You can also set superadmin in the users COLUMN of the DATABASE. I haven't figured out what this is for though.

You shoud now be able to play around with your API here:
[https://gotruejs-playground.netlify.com/](https://gotruejs-playground.netlify.com/)

With the [gotrue-js](https://github.com/netlify/gotrue-js) library, as a Javascript-Developer, you could easily develop your own gotrue-admin-page. My Javascript is poor, thats why I kept talking to the api with [Imsomnia](https://insomnia.rest/): a REST Client for debugging/understanding API's - made for humans.

### Adding a new User via API (Insomnia etc.)

Create a Post Request to ```<api-root>/admin/users``` :

Body:
```json
{
  "email":     "admin@example.com",
  "password":  "password",
  "role":      "admin"
}
```
And add a Bearer Token with the "super-secret-operator-token" (You have to set it in your configuration). And send it. 

Hitting ```<api-root>/admin/users``` with a GET Request and the Operator Token as authentification you'll get a list of all registered users in your response.

### Editing the Database directly
I haven't tried this at first, because before I found out above I logged into mariadb directly and gave myself the admin role like this:

```
mysql> USE DATABASE gotrue;
mysql> UPDATE users SET role='admin' WHERE email='email@example.org';
```

These are roles for gotrue. Don't confuse them with the roles of netlify-cms or git-gateway.
They are set in the app_metadata field. 

This is what a user object in gotrue looks like:


```json
{
  "id": "5d6ca16d-a63a-43e9-9447-89ed6712df05",
  "aud": "",
  "role": "admin",
  "email": "email@example.com",
  "confirmed_at": "2018-12-22T13:56:56Z",
  "app_metadata": {
    "provider": "email"
  },
  "user_metadata": {
    "name": "Max Mustermann",
    "roles": [ "admin", "editor" ]
  },
  "created_at": "2018-12-22T13:56:54Z",
  "updated_at": "2018-12-22T13:56:54Z"
}
```

## Make it work with Netlify CMS

Easy Setup is using the [netlify-identity widget](https://github.com/netlify/netlify-identity-widget).

the index.html for the example.com/admin page migth look like this:

```html
<!doctype html>
<html>
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0" />
  <title>Content Manager</title>
  <script src="netlify-identity-widget.js"></script>
</head>
<body>
  <script src="netlify-cms.js"></script>
  <script>
  if (window.netlifyIdentity) {
    window.netlifyIdentity.on("init", user => {
      if (!user) {
        window.netlifyIdentity.on("login", () => {
          document.location.href = "/admin/";
        });
      }
    });
  }
</script>
</body>
</html>
```
You can change the redirect point after an successful login in this line:

```
document.location.href = "/admin/
```

I'm still not clear about the differences between the netlify-identity-widget.js and netlify-identity.js . P

and the config.yml might start like this:

```yaml
backend:
  name: git-gateway
  branch: master
  identity_url: "https://example.com/.netlify/identity"
  gateway_url:  "https://example.com/.netlify/git"
  site_domain:  "https://example.com/"
  accept_roles: #optional - accepts all users if left out
    - admin
    - editor
```

The .netlify/identity and .netlify/git paths are the default ones netlify cms and the identity widgets look for. You have to setup a proxy in your webservers config in order to reach the APIs of gotrue and git-gateway with these links.

Nginx example, make sure you use https in the end:

```
upstream gotrue {
  server localhost:8081;
}	

upstream gitgateway {
  server localhost:9999;
}

server {

  listen       80;
  server_name  example.com;
  #
  # location / {
  #   root /root_of_your_site;
  #  index index.html;
  # }

  location /.netlify/identity/ {
    proxy_redirect   off;
    proxy_set_header Host              $http_host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://gotrue/;
  }

  location /.netlify/git/ {
    proxy_redirect   off;
    proxy_set_header Host              $http_host;
    proxy_set_header X-Real-IP         $remote_addr;
    proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_pass http://gitgateway/;
  }

}
```

With caddy-webserver this comes in even more handy:
```
example.com {
	root /root_of_website
	proxy /.netlify/identity localhost:9998 {
		without /.netlify/identity
        transparent
	}
	proxy /.netlify/git localhost:9999 {
		without /.netlify/git
		transparent
	}
}
```


## Conclusion

Please feel free to contact me if you have any input on this tutorial. This is like a first draft.

