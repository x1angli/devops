# 在群晖上利用 acme.sh 安装和自动更新 SSL 证书

## 前言

笔者曾在 [另一篇文章](https://github.com/x1angli/devops/blob/master/install-https-certificate-on-synology.md) 中提到 如何在本地客户端上通过 [Certbot](https://certbot.eff.org/) 的 DNS Challenge 手动生成 HTTPS/SSL 证书并部署至群晖 Synology。之所以那么大费周张，原因：1) 最主要的是，ISP **阻断了 web 80 端口**（当然，其他常用的若干端口也被阻断了），导致无法用群晖内置的 Let's Encrypt 证书申请及更新模块，（其实如果ISP能开放80端口能节省很多折腾）；2) 当时貌似没有在群晖的 Terminal 上调用其他的客户端，只能用最原始的命令。

时过境迁，目前笔者给自己提出了更高一层的任务。需求端方面，Let's Encrypt 颁发证书只有有限的三个月，因此需要频繁地手动更新，不但繁琐，而且极易造成因证书过期导致的服务中断或者异常。而供给端方面，笔者这段时间也接触了更多的SSL证书自更新的客户端，同时这些客户端也推出了层出不穷的新功能。因此，笔者现在将通过 [acme.sh](https://acme.sh/) 实现在服务器端的证书创建、以及**自动更新** auto-renew 。

## 步骤

### 1. 安装 acme.sh

首先要在群晖的控制面板的设置里开通 SSH，并且确保防火墙的相关设置没有阻塞。用 SSH 登录到群晖的 Terminal，并执行如下脚本

```sh
sudo -i 
# Put in your admin password here so as to escalate to the "root" user

wget https://github.com/Neilpang/acme.sh/archive/master.tar.gz
tar xvf master.tar.gz
cd acme.sh-master/

./acme.sh --install --nocron --home /usr/local/share/acme.sh --accountemail "<youradmin@yourdomain.tld>"
```

注意：由于群晖有比较严格的账号系统，故而要提权至root。

### 2. 首次创建并部署证书

```sh
sudo -i

export CERT_FOLDER="/usr/syno/etc/certificate/_archive/<random_hex>"
export NAMECHEAP_USERNAME="<namecheap_username>"
export NAMECHEAP_API_KEY="<namecheap_api_key_hex>"
export NAMECHEAP_SOURCEIP="<your_synology_egress_IP>"

cd /usr/local/share/acme.sh

./acme.sh --issue --dns dns_namecheap --force --debug \
    -d <subdomain1.yourdomain.tld> -d <subdomain2.yourdomain.tld> \
    --cert-file "$CERT_FOLDER/cert.pem" \
    --key-file "$CERT_FOLDER/privkey.pem" \
    --fullchain-file "$CERT_FOLDER/fullchain.pem" \
	--reloadcmd "rsync -avh $CERT_FOLDER/ /usr/syno/etc/certificate/system/default && /usr/syno/sbin/synoservicectl --reload nginx" \
    --dnssleep 60
```

注意：

1. 最关键的：群晖的证书，目前存在两处位置，一处是 `/usr/syno/etc/certificate/_archive/<random_hex>` 其中 `<random_hex>` 的值，你需要运行 `cat /usr/syno/etc/certificate/_archive/default`  来获取到。这里的证书是控制面板里的证书，通过将生成的证书放在这儿，就意味着将覆盖群晖里现有的已经被设为“default”的默认证书。另一处是 `/usr/syno/etc/certificate/system/default` 此处是 DSM 管理界面的证书。笔者建议两处证书保持一致（如同这里通过`rsync`命令同步一样）
2. 同样关键的：由于 acme.sh 默认不建议在 root 角色下执行，但由于群晖默认的严权限，导致必须在 root 下运行，因此需要 `--force` 选项（这的确不优美，无奈……）
3. 这段脚本是基于 NameCheap 的体系。无论你用的是 NameCheap 还是其他，请参考 [HTTPS certificates for your Synology NAS using acme.sh](https://github.com/Neilpang/acme.sh/wiki/Synology-NAS-Guide) 一文
4. `NAMECHEAP_SOURCEIP` 处填写当前群晖的出口的公网 IP。貌似如果你已经在 NameCheap 的 API 白名单里手动添加的话，这里貌似就无所谓了
5. `dnssleep` 本处设置成60（即60秒），请酌情设置。如果你的 DNS 的更新能比较快被捕捉到，可以设置得短一些；否则需要设得久一些。
6. 其他，请酌情修改
7. 由于国际网络，有可能造成访问不畅，这时候需要多试几次。

### 3. 设置自动更新

在 `/root` 下新建一个文件 ，暂且命名为 `renew_cert.sh` ，填入如下内容

```sh
#!/usr/bin/env sh

CERT_FOLDER="/usr/syno/etc/certificate/_archive/<random_hex>"

/usr/local/share/acme.sh/acme.sh --cron --force --debug

rsync -avh $CERT_FOLDER/ /usr/syno/etc/certificate/system/default
/usr/syno/sbin/synoservicectl --reload nginx

PACKAGECERTROOTDIR="/usr/local/etc/certificate"

# update and restart all installed packages
PEMFILES=$(find $PACKAGECERTROOTDIR -name cert.pem)
if [ ! -z "$PEMFILES" ]; then
	for DIR in $PEMFILES; do
		#active directory has it's own certificate so we do not update that package
		if [[ $DIR != *"/ActiveDirectoryServer/"* ]]; then
			rsync -avh "$CERT_FOLDER/" "$(dirname $DIR)/"
			/usr/syno/bin/synopkg restart $(echo $DIR | awk -F/ '{print $6}')
		fi
	done
fi
```

打开 `控制面板 / 任务计划 / 新增 / 计划的任务 / 用户自定义的脚本`
![计划的任务 -> 用户自定义的脚本](https://up4dev.oss-cn-qingdao.aliyuncs.com/nas-cert-up/task.png)]

用 `root` 账号，设置一个“每月重复”

## 后记

* acme.sh 在申请证书时，会在服务器上保存 DNS 服务商的 key，可能会有一定的安全隐患，不过没有更好的办法 
* Alpine 协议目前还不成熟，易用性不强，尝试以失败告终

## 参考文献

1. [指南攻略: 在群晖上安装SSL证书](https://github.com/x1angli/devops/blob/master/install-https-certificate-on-synology.md)
2. [HTTPS certificates for your Synology NAS using acme.sh](https://github.com/Neilpang/acme.sh/wiki/Synology-NAS-Guide)
3. [群晖 Let's Encrypt 泛域名证书自动更新](http://www.up4dev.com/2018/05/29/synology-ssl-wildcard-cert-update/)
4. [群晖 Let's Encrypt 证书的自动更新](http://www.up4dev.com/2017/09/11/synology-ssl-cert-update/)
