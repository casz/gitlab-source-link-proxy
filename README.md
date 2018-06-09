# GitLab configuration

Configure GitLab nginx to proxy the Visual Studio requests through our proxy:

```
vim /var/opt/gitlab/nginx/conf/gitlab-http.conf 
...
  location / {
    if ($http_user_agent ~ "SourceLink") {
      #proxy_set_header Authorization "Basic cm9vdDpwYXNzd29yZA==";
      proxy_pass http://127.0.0.1:7000;
      break;
    }
    proxy_cache off;
    proxy_pass  http://gitlab-workhorse;
  }

gitlab-ctl restart nginx
```

Build this binary:

```bash
go get -u -v github.com/jamiealquiza/bicache
go get -u -v golang.org/x/crypto/blake2b
go build -v
```

Manually run it:

```bash
./gitlab-source-link-proxy --gitlab-base-url https://gitlab.example.com
```

Try it:

```bash
http --verify=no -v https://root:password@gitlab.example.com/root/ubuntu-vagrant/raw/master/.gitignore User-Agent:SourceLink
```

Install it as a systemd service:

```bash
# add the gitlab-source-link-proxy user.
groupadd --system gitlab-source-link-proxy
adduser \
    --system \
    --disabled-login \
    --no-create-home \
    --gecos '' \
    --ingroup gitlab-source-link-proxy \
    --home /opt/gitlab-source-link-proxy \
    gitlab-source-link-proxy
install -d -o root -g gitlab-source-link-proxy -m 750 /opt/gitlab-source-link-proxy
install -d -o root -g gitlab-source-link-proxy -m 750 /opt/gitlab-source-link-proxy/bin

# install the binary.
install -o root -g root -m 555 gitlab-source-link-proxy /opt/gitlab-source-link-proxy/bin

# install the systemd service.
install -o root -g root -m 644 gitlab-source-link-proxy.service /etc/systemd/system/
systemctl enable gitlab-source-link-proxy
systemctl start gitlab-source-link-proxy
```

Show the service status and logs:

```bash
systemctl status gitlab-source-link-proxy
journalctl -u gitlab-source-link-proxy
```

# Visual Studio configuration

To be able to step through external source code you need to disable `Enable Just My Code` setting in the `Tools | Options | Debugger` window.

To be able to download source code you need to configure Visual Studio to authenticate your GitLab domain requests:

  1. Open the `Developer Command Prompt for VS 2017`
  2. run `notepad "%VSINSTALLDIR%\Common7\IDE\CommonExtensions\Platform\Debugger\VsDebugPresentationPackage.pkgdef"`
  3. Add your GitLab domain as a Git Credential Manager Authority in the corresponding ini section, e.g.:
      ```ini
      [$RootKey$\Debugger\GitCredentialManager\Authorities]
      "raw.githubusercontent.com"="https://github.com"
      "gitlab.example.com"="https://gitlab.example.com"
      ```
  4. run `devenv /setup`
