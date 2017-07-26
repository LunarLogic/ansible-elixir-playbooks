# Ansible Elixir Playbooks

This project has ansible playbooks for:

* Elixr Build Server - it has installed Erlang, Elixir and nodejs. Basically, all what is required to compile the Phoenix Framework application.
* Phoenix Website - it is a playbook to provision server with installed PostgreSQL and configured nginx and Let's Encrypt for SSL. There is no Erlang/Elixir on this server because we will deploy there only compiled Phoenix application.

You can learn more about the project from this blog post (TODO: add here url).

Here you will find an example [Phoenix Framework app configured for deployment](https://github.com/LunarLogic/phoenix_website).

## Requirements

### Control machine (your computer)

* Install [Ansible](https://www.ansible.com/)

* Download roles:

  ```shell
  $ ansible-galaxy install -r requirements.yml
  ```

* Generate `vault_pass.txt` file into this repository. You need it to be able to encrypt/decrypt secrets.

  :warning: For security reasons, the `vault_pass.txt` file should not be committed into the repository. It's ignored in `.gitignore`.

  ```shell
  $ openssl rand -base64 256 > vault_pass.txt
  ```

* Generate your DB password and put output to `apps/phoenix-website/host_vars/phoenix-website.lunarlogic.io`

  ```shell
  $ ansible-vault encrypt_string --name db_password "YOUR_DB_PASSWORD"
  ```

### Target machine (server)

* Server with [Ubuntu](https://www.ubuntu.com/) 16.04 LTS.

## Public keys

We keep our public keys in [public_keys/] directory. This set of keys is uploaded to the server during each provisioning
and overwrites the list of authorized keys, so proper people have access to the server. It is important to keep
the list of keys up to date.

If you don't know how to generate a key for yourself,
[read this article](https://help.github.com/articles/connecting-to-github-with-ssh/).

## App deployement

### CircleCI deployment

If you want to deploy app from CI to the staging/production host then you must generate RSA keys for CircleCI.

```shell
$ ssh-keygen -t rsa -b 4096 -N "" -C "circle_ci" -f ./apps/elixir-build-server/circle_ci
$ ssh-keygen -t rsa -b 4096 -N "" -C "circle_ci" -f ./apps/phoenix-website/circle_ci
```

Add `circle_ci.pub` public key to your app playbook for the role `user`:

```
- role: user/0.0.1
  username: phoenix
  authorized_key_paths:
    - ../../public_keys/*.pub
    - ./circle_ci.pub # add this line
```

Go to CircleCI and find your project, open settings and find `SSH Permissions`. Click `Add an SSH key` button and paste there private key `apps/YOUR_APP_NAME/circle_ci`.

Now you can remove private key `apps/YOUR_APP_NAME/circle_ci` from local machine. It should not be commited into repo!

Commit into repo only public key `apps/YOUR_APP_NAME/circle_ci.pub`.

You can always generate a new fresh keys if you need it hence no reason to backup private key. You already added it to CircleCI.

## Run playbooks

__Warning:__ This command will provision all servers listed in inventory file for particular app `apps/app_name`.

```shell
$ ./play apps/app_name
```

If you want to provision only specific machine do (it's useful if your app is deployed to multiple servers like staging and production):

```shell
# Warning: There must be comma and the end of the hosts list!
$ ansible-playbook -i 'example-staging.lunarlogic.io,' apps/app_name/playbook.yml
```

## Provisioning logs

You can check when and with what git commit the host was provisioned in log file: `/var/log/provision.log` (stored on the target machine).

## System users

There are 3 types of users on the server:

* `root` - for provisioning
* `admin` - user has the sudo access
* `app_name_user` - for instance `phoenix` user for Phoenix Website application. The user has no sudo access. The application is running under this user.

## Add playbook for new app

* create new app directory in the `app` directory
* in this new directory create `playbook.yml` and `inventory` files
* in the `inventory` file put host names to provision (see [Ansible docs](http://docs.ansible.com/ansible/intro_inventory.html))
* implement `playbook.yml`

## Secrets

We store secrets in encrypted version using [Vault]. If you are adding new secrets, make sure you commit them to the repository in the encrypted form.

* Encrypting single values (that can be placed inside a "clear text" YAML file, using the `!vault` tag):

  ```shell
  $ ansible-vault encrypt_string --name pass_to_some_service "secret"  # stdout encrypted string
  ```

* Encrypting whole YAML files:

  ```shell
  $ ansible-vault encrypt secret.yml   # encrypt unencrypted file
  $ ansible-vault edit secret.yml      # edit encrypted file
  $ ansible-vault decrypt secret.yml   # decrypt encrypted file
  ```

## Roles

### Role versioning

We use roles versioning the simplest possible way, we just add version subdirectories under every role directory.

```shell
roles/role-name/role-version/ # e.g. roles/webserver/0.3.2/
```

To create a new version just copy an existing one, bump the role version and modify it.
Please, respect [Semantic Versioning 2.0.0].

### Community developed roles

Include the roles in [requirements.yml] and download them using the following command:

```shell
$ ansible-galaxy install -r requirements.yml
```

### SSL with Let's Encrypt

You can use `lets_encrypt` role to generate free SSL certificate thanks to https://letsencrypt.org

#### Rate Limits

The main limit is Certificates per Registered Domain (20 per week).

https://letsencrypt.org/docs/rate-limits/

If you are testing Let's Encrypt then use `staging` environment with higher limits!

```
- role: lets_encrypt/0.0.1
  app_name: myapp
  lets_encrypt_contact_email: admin@lunarlogic.io
  lets_encrypt_environment: staging # you can change it to production once ready
```

#### If you want to change main domain for your certificate

If you want to change main domain for your certificate then you need to generate a new certificate.

Here is example file for [Phoenix Website project with multiple domains](apps/phoenix-website/host_vars/phoenix-website.lunarlogic.io).

In order to generate a new certificate please remove first the old files generated by `lets_encrypt` role on the server:

```shell
$ rm -rf /etc/letsencrypt/accounts/*
$ rm -rf /etc/letsencrypt/archive/*
$ rm -rf /etc/letsencrypt/csr/*
$ rm -rf /etc/letsencrypt/keys/*
$ rm -rf /etc/letsencrypt/live/*
$ rm -rf /etc/letsencrypt/renewal/*

# remove the snippents that load SSL certificate
$ rm -rf /etc/nginx/snippets/project_name
```

Ensure the nginx is running. It's required so Let's Encrypt can do request to our domain.
Provision server again.

Note: If you would like to add a new subdomain to domain list then you can just provision server and a new subdomain will be added to the certificate. You need to generate certificate from scrach only if you change the main domain (the first domain on the list of domains).


[Vault]: http://docs.ansible.com/ansible/playbooks_vault.html
[public_keys/]: public_keys/
[requirements.yml]: requirements.yml
[Semantic Versioning 2.0.0]: http://semver.org/
