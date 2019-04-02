title: 实验文档2：交付一套dubbo微服务到kubernetes集群
author: Stanley Wang
categories: Kubernetes容器云专题
date: 2019-1-16 21:12:56
---
# 基础架构
主机名|角色|ip
-|-|-
HDSS7-11.host.com|k8s代理节点1，zk1|10.4.7.11
HDSS7-12.host.com|k8s代理节点2，zk2|10.4.7.12
HDSS7-21.host.com|k8s运算节点1，zk3|10.4.7.21
HDSS7-22.host.com|k8s运算节点2，jenkins|10.4.7.22
HDSS7-200.host.com|k8s运维节点(docker仓库)|10.4.7.200

# 部署zookeeper
## 安装jdk1.7（3台zk角色主机）
> jdk下载地址
> [jdk1.6](https://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase6-419409.html)
> [jdk1.7](https://www.oracle.com/technetwork/java/javase/downloads/java-archive-downloads-javase7-521261.html)
> [jdk1.8](https://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)

```pwd /opt/src
[root@hdss7-11 src]# ls -l|grep jdk
-rw-r--r-- 1 root root 153530841 Jan 17 17:49 jdk-8u201-linux-x64.tar.gz
[root@hdss7-11 src]# mkdir /usr/java
[root@hdss7-11 src]# tar xf jdk-8u201-linux-x64.tar.gz -C /usr/java
[root@hdss7-11 src]# ln -s /usr/java/jdk1.8.0_201 /usr/java/jdk
[root@hdss7-11 src]# vi /etc/profile
export JAVA_HOME=/usr/java/jdk
export PATH=$JAVA_HOME/bin:$JAVA_HOME/bin:$PATH
export CLASSPATH=$CLASSPATH:$JAVA_HOME/lib:$JAVA_HOME/lib/tools.jar
```
## 安装zookeeper（3台zk角色主机）
> zk下载地址
> [zookeeper](https://archive.apache.org/dist/zookeeper/)

```pwd /opt/src
[root@hdss7-11 src]# ls -l|grep zoo
-rw-r--r-- 1 root root 153530841 Jan 17 18:10 zookeeper-3.4.14.tar.gz
[root@hdss7-11 src]#
没完
```

# 部署jenkins
## 准备镜像
> [jenkins官网](https://jenkins.io/download/)
> [jenkins镜像](https://hub.docker.com/r/jenkins/jenkins)

在运维主机下载官网上的稳定版（这里下载2.164.1）
```
[root@hdss7-200 ~]#  docker pull jenkins/jenkins:2.164.1
2.164.1: Pulling from jenkins/jenkins
22dbe790f715: Pull complete 
0250231711a0: Pull complete 
6fba9447437b: Pull complete 
c2b4d327b352: Pull complete 
cddb9bb0d37c: Pull complete 
b535486c968f: Pull complete 
f3e976e6210c: Pull complete 
b2c11b10291d: Pull complete 
f4c0181e1976: Pull complete 
924c8e712392: Pull complete 
d13006b7c9dd: Pull complete 
fc80aeb92627: Pull complete 
36a6e96ba1b5: Pull complete 
f50f33dc1d0a: Pull complete 
b10642432117: Pull complete 
850c260511d8: Pull complete 
47f95e65a629: Pull complete 
3b33ce546dc6: Pull complete 
051c7665e760: Pull complete 
fe379aecc538: Pull complete 
Digest: sha256:12fd14965de7274b5201653b2bffa62700c5f5f336ec75c945321e2cb70d7af0
Status: Downloaded newer image for jenkins/jenkins:2.164.1

[root@hdss7-200 ~]# docker tag jenkins/jenkins:2.164.1 harbor.od.com/public/jenkins:2.164.1
[root@hdss7-200 ~]# docker push !$
```
## 自定义Dockerfile
在运维主机`HDSS7-200.host.com`上编辑自定义dockerfile
```vi /data/dockerfile/jenkins/Dockerfile
FROM harbor.od.com/public/jenkins:2.164.1
RUN ln -s /usr/share/zoneinfo/Asia/Shanghai /etc/localtime &&\ 
    echo 'Asia/Shanghai' >/etc/timezone
USER root
ADD config.json /root/.docker/config.json
ADD get-docker.sh /get-docker.sh
RUN /get-docker.sh
```
这个Dockerfile里我们主要做了以下几件事
- 设置容器内的时区
- 设置容器用户为root
- 加入了登录自建harbor仓库的config文件
- 加入了一个get-docker的shell脚本并运行

{% tabs jenkins%}
<!-- tab config.json -->
{% code %}
{
	"auths": {
		"harbor.od.com": {
			"auth": "YWRtaW46SGFyYm9yMTIzNDU="
		}
	}
}
{% endcode %}
<!-- endtab -->
<!-- tab get-docker.sh-->
{% code %}
#!/bin/sh
set -e

# This script is meant for quick & easy install via:
#   $ curl -fsSL get.docker.com -o get-docker.sh
#   $ sh get-docker.sh
#
# For test builds (ie. release candidates):
#   $ curl -fsSL test.docker.com -o test-docker.sh
#   $ sh test-docker.sh
#
# NOTE: Make sure to verify the contents of the script
#       you downloaded matches the contents of install.sh
#       located at https://github.com/docker/docker-install
#       before executing.
#
# Git commit from https://github.com/docker/docker-install when
# the script was uploaded (Should only be modified by upload job):
SCRIPT_COMMIT_SHA=36b78b2


# This value will automatically get changed for:
#   * edge
#   * test
#   * experimental
DEFAULT_CHANNEL_VALUE="edge"
if [ -z "$CHANNEL" ]; then
	CHANNEL=$DEFAULT_CHANNEL_VALUE
fi

DEFAULT_DOWNLOAD_URL="https://download.docker.com"
if [ -z "$DOWNLOAD_URL" ]; then
	DOWNLOAD_URL=$DEFAULT_DOWNLOAD_URL
fi

DEFAULT_REPO_FILE="docker-ce.repo"
if [ -z "$REPO_FILE" ]; then
	REPO_FILE="$DEFAULT_REPO_FILE"
fi

SUPPORT_MAP="
x86_64-centos-7
x86_64-fedora-26
x86_64-fedora-27
x86_64-fedora-28
x86_64-debian-wheezy
x86_64-debian-jessie
x86_64-debian-stretch
x86_64-debian-buster
x86_64-ubuntu-trusty
x86_64-ubuntu-xenial
x86_64-ubuntu-bionic
x86_64-ubuntu-artful
s390x-ubuntu-xenial
s390x-ubuntu-bionic
s390x-ubuntu-artful
ppc64le-ubuntu-xenial
ppc64le-ubuntu-bionic
ppc64le-ubuntu-artful
aarch64-ubuntu-xenial
aarch64-ubuntu-bionic
aarch64-debian-jessie
aarch64-debian-stretch
aarch64-debian-buster
aarch64-fedora-26
aarch64-fedora-27
aarch64-fedora-28
aarch64-centos-7
armv6l-raspbian-jessie
armv7l-raspbian-jessie
armv6l-raspbian-stretch
armv7l-raspbian-stretch
armv7l-debian-jessie
armv7l-debian-stretch
armv7l-debian-buster
armv7l-ubuntu-trusty
armv7l-ubuntu-xenial
armv7l-ubuntu-bionic
armv7l-ubuntu-artful
"

mirror=''
DRY_RUN=${DRY_RUN:-}
while [ $# -gt 0 ]; do
	case "$1" in
		--mirror)
			mirror="$2"
			shift
			;;
		--dry-run)
			DRY_RUN=1
			;;
		--*)
			echo "Illegal option $1"
			;;
	esac
	shift $(( $# > 0 ? 1 : 0 ))
done

case "$mirror" in
	Aliyun)
		DOWNLOAD_URL="https://mirrors.aliyun.com/docker-ce"
		;;
	AzureChinaCloud)
		DOWNLOAD_URL="https://mirror.azure.cn/docker-ce"
		;;
esac

command_exists() {
	command -v "$@" > /dev/null 2>&1
}

is_dry_run() {
	if [ -z "$DRY_RUN" ]; then
		return 1
	else
		return 0
	fi
}

deprecation_notice() {
	distro=$1
	date=$2
	echo
	echo "DEPRECATION WARNING:"
	echo "    The distribution, $distro, will no longer be supported in this script as of $date."
	echo "    If you feel this is a mistake please submit an issue at https://github.com/docker/docker-install/issues/new"
	echo
	sleep 10
}

get_distribution() {
	lsb_dist=""
	# Every system that we officially support has /etc/os-release
	if [ -r /etc/os-release ]; then
		lsb_dist="$(. /etc/os-release && echo "$ID")"
	fi
	# Returning an empty string here should be alright since the
	# case statements don't act unless you provide an actual value
	echo "$lsb_dist"
}

add_debian_backport_repo() {
	debian_version="$1"
	backports="deb http://ftp.debian.org/debian $debian_version-backports main"
	if ! grep -Fxq "$backports" /etc/apt/sources.list; then
		(set -x; $sh_c "echo \"$backports\" >> /etc/apt/sources.list")
	fi
}

echo_docker_as_nonroot() {
	if is_dry_run; then
		return
	fi
	if command_exists docker && [ -e /var/run/docker.sock ]; then
		(
			set -x
			$sh_c 'docker version'
		) || true
	fi
	your_user=your-user
	[ "$user" != 'root' ] && your_user="$user"
	# intentionally mixed spaces and tabs here -- tabs are stripped by "<<-EOF", spaces are kept in the output
	echo "If you would like to use Docker as a non-root user, you should now consider"
	echo "adding your user to the \"docker\" group with something like:"
	echo
	echo "  sudo usermod -aG docker $your_user"
	echo
	echo "Remember that you will have to log out and back in for this to take effect!"
	echo
	echo "WARNING: Adding a user to the \"docker\" group will grant the ability to run"
	echo "         containers which can be used to obtain root privileges on the"
	echo "         docker host."
	echo "         Refer to https://docs.docker.com/engine/security/security/#docker-daemon-attack-surface"
	echo "         for more information."

}

# Check if this is a forked Linux distro
check_forked() {

	# Check for lsb_release command existence, it usually exists in forked distros
	if command_exists lsb_release; then
		# Check if the `-u` option is supported
		set +e
		lsb_release -a -u > /dev/null 2>&1
		lsb_release_exit_code=$?
		set -e

		# Check if the command has exited successfully, it means we're in a forked distro
		if [ "$lsb_release_exit_code" = "0" ]; then
			# Print info about current distro
			cat <<-EOF
			You're using '$lsb_dist' version '$dist_version'.
			EOF

			# Get the upstream release info
			lsb_dist=$(lsb_release -a -u 2>&1 | tr '[:upper:]' '[:lower:]' | grep -E 'id' | cut -d ':' -f 2 | tr -d '[:space:]')
			dist_version=$(lsb_release -a -u 2>&1 | tr '[:upper:]' '[:lower:]' | grep -E 'codename' | cut -d ':' -f 2 | tr -d '[:space:]')

			# Print info about upstream distro
			cat <<-EOF
			Upstream release is '$lsb_dist' version '$dist_version'.
			EOF
		else
			if [ -r /etc/debian_version ] && [ "$lsb_dist" != "ubuntu" ] && [ "$lsb_dist" != "raspbian" ]; then
				if [ "$lsb_dist" = "osmc" ]; then
					# OSMC runs Raspbian
					lsb_dist=raspbian
				else
					# We're Debian and don't even know it!
					lsb_dist=debian
				fi
				dist_version="$(sed 's/\/.*//' /etc/debian_version | sed 's/\..*//')"
				case "$dist_version" in
					9)
						dist_version="stretch"
					;;
					8|'Kali Linux 2')
						dist_version="jessie"
					;;
					7)
						dist_version="wheezy"
					;;
				esac
			fi
		fi
	fi
}

semverParse() {
	major="${1%%.*}"
	minor="${1#$major.}"
	minor="${minor%%.*}"
	patch="${1#$major.$minor.}"
	patch="${patch%%[-.]*}"
}

ee_notice() {
	echo
	echo
	echo "  WARNING: $1 is now only supported by Docker EE"
	echo "           Check https://store.docker.com for information on Docker EE"
	echo
	echo
}

do_install() {
	echo "# Executing docker install script, commit: $SCRIPT_COMMIT_SHA"

	if command_exists docker; then
		docker_version="$(docker -v | cut -d ' ' -f3 | cut -d ',' -f1)"
		MAJOR_W=1
		MINOR_W=10

		semverParse "$docker_version"

		shouldWarn=0
		if [ "$major" -lt "$MAJOR_W" ]; then
			shouldWarn=1
		fi

		if [ "$major" -le "$MAJOR_W" ] && [ "$minor" -lt "$MINOR_W" ]; then
			shouldWarn=1
		fi

		cat >&2 <<-'EOF'
			Warning: the "docker" command appears to already exist on this system.

			If you already have Docker installed, this script can cause trouble, which is
			why we're displaying this warning and provide the opportunity to cancel the
			installation.

			If you installed the current Docker package using this script and are using it
		EOF

		if [ $shouldWarn -eq 1 ]; then
			cat >&2 <<-'EOF'
			again to update Docker, we urge you to migrate your image store before upgrading
			to v1.10+.

			You can find instructions for this here:
			https://github.com/docker/docker/wiki/Engine-v1.10.0-content-addressability-migration
			EOF
		else
			cat >&2 <<-'EOF'
			again to update Docker, you can safely ignore this message.
			EOF
		fi

		cat >&2 <<-'EOF'

			You may press Ctrl+C now to abort this script.
		EOF
		( set -x; sleep 20 )
	fi

	user="$(id -un 2>/dev/null || true)"

	sh_c='sh -c'
	if [ "$user" != 'root' ]; then
		if command_exists sudo; then
			sh_c='sudo -E sh -c'
		elif command_exists su; then
			sh_c='su -c'
		else
			cat >&2 <<-'EOF'
			Error: this installer needs the ability to run commands as root.
			We are unable to find either "sudo" or "su" available to make this happen.
			EOF
			exit 1
		fi
	fi

	if is_dry_run; then
		sh_c="echo"
	fi

	# perform some very rudimentary platform detection
	lsb_dist=$( get_distribution )
	lsb_dist="$(echo "$lsb_dist" | tr '[:upper:]' '[:lower:]')"

	case "$lsb_dist" in

		ubuntu)
			if command_exists lsb_release; then
				dist_version="$(lsb_release --codename | cut -f2)"
			fi
			if [ -z "$dist_version" ] && [ -r /etc/lsb-release ]; then
				dist_version="$(. /etc/lsb-release && echo "$DISTRIB_CODENAME")"
			fi
		;;

		debian|raspbian)
			dist_version="$(sed 's/\/.*//' /etc/debian_version | sed 's/\..*//')"
			case "$dist_version" in
				9)
					dist_version="stretch"
				;;
				8)
					dist_version="jessie"
				;;
				7)
					dist_version="wheezy"
				;;
			esac
		;;

		centos)
			if [ -z "$dist_version" ] && [ -r /etc/os-release ]; then
				dist_version="$(. /etc/os-release && echo "$VERSION_ID")"
			fi
		;;

		rhel|ol|sles)
			ee_notice "$lsb_dist"
			exit 1
			;;

		*)
			if command_exists lsb_release; then
				dist_version="$(lsb_release --release | cut -f2)"
			fi
			if [ -z "$dist_version" ] && [ -r /etc/os-release ]; then
				dist_version="$(. /etc/os-release && echo "$VERSION_ID")"
			fi
		;;

	esac

	# Check if this is a forked Linux distro
	check_forked

	# Check if we actually support this configuration
	if ! echo "$SUPPORT_MAP" | grep "$(uname -m)-$lsb_dist-$dist_version" >/dev/null; then
		cat >&2 <<-'EOF'

		Either your platform is not easily detectable or is not supported by this
		installer script.
		Please visit the following URL for more detailed installation instructions:

		https://docs.docker.com/engine/installation/

		EOF
		exit 1
	fi

	# Run setup for each distro accordingly
	case "$lsb_dist" in
		ubuntu|debian|raspbian)
			pre_reqs="apt-transport-https ca-certificates curl"
			if [ "$lsb_dist" = "debian" ]; then
				if [ "$dist_version" = "wheezy" ]; then
					add_debian_backport_repo "$dist_version"
				fi
				# libseccomp2 does not exist for debian jessie main repos for aarch64
				if [ "$(uname -m)" = "aarch64" ] && [ "$dist_version" = "jessie" ]; then
					add_debian_backport_repo "$dist_version"
				fi
			fi

			# TODO: August 31, 2018 delete from here,
			if [ "$lsb_dist" =  "ubuntu" ] && [ "$dist_version" = "artful" ]; then
				deprecation_notice "$lsb_dist $dist_version" "August 31, 2018"
			fi
			# TODO: August 31, 2018 delete to here,

			if ! command -v gpg > /dev/null; then
				pre_reqs="$pre_reqs gnupg"
			fi
			apt_repo="deb [arch=$(dpkg --print-architecture)] $DOWNLOAD_URL/linux/$lsb_dist $dist_version $CHANNEL"
			(
				if ! is_dry_run; then
					set -x
				fi
				$sh_c 'apt-get update -qq >/dev/null'
				$sh_c "apt-get install -y -qq $pre_reqs >/dev/null"
				$sh_c "curl -fsSL \"$DOWNLOAD_URL/linux/$lsb_dist/gpg\" | apt-key add -qq - >/dev/null"
				$sh_c "echo \"$apt_repo\" > /etc/apt/sources.list.d/docker.list"
				if [ "$lsb_dist" = "debian" ] && [ "$dist_version" = "wheezy" ]; then
					$sh_c 'sed -i "/deb-src.*download\.docker/d" /etc/apt/sources.list.d/docker.list'
				fi
				$sh_c 'apt-get update -qq >/dev/null'
			)
			pkg_version=""
			if [ ! -z "$VERSION" ]; then
				if is_dry_run; then
					echo "# WARNING: VERSION pinning is not supported in DRY_RUN"
				else
					# Will work for incomplete versions IE (17.12), but may not actually grab the "latest" if in the test channel
					pkg_pattern="$(echo "$VERSION" | sed "s/-ce-/~ce~.*/g" | sed "s/-/.*/g").*-0~$lsb_dist"
					search_command="apt-cache madison 'docker-ce' | grep '$pkg_pattern' | head -1 | cut -d' ' -f 4"
					pkg_version="$($sh_c "$search_command")"
					echo "INFO: Searching repository for VERSION '$VERSION'"
					echo "INFO: $search_command"
					if [ -z "$pkg_version" ]; then
						echo
						echo "ERROR: '$VERSION' not found amongst apt-cache madison results"
						echo
						exit 1
					fi
					pkg_version="=$pkg_version"
				fi
			fi
			(
				if ! is_dry_run; then
					set -x
				fi
				$sh_c "apt-get install -y -qq --no-install-recommends docker-ce$pkg_version >/dev/null"
			)
			echo_docker_as_nonroot
			exit 0
			;;
		centos|fedora)
			yum_repo="$DOWNLOAD_URL/linux/$lsb_dist/$REPO_FILE"
			if ! curl -Ifs "$yum_repo" > /dev/null; then
				echo "Error: Unable to curl repository file $yum_repo, is it valid?"
				exit 1
			fi
			if [ "$lsb_dist" = "fedora" ]; then
				if [ "$dist_version" -lt "26" ]; then
					echo "Error: Only Fedora >=26 are supported"
					exit 1
				fi

				pkg_manager="dnf"
				config_manager="dnf config-manager"
				enable_channel_flag="--set-enabled"
				pre_reqs="dnf-plugins-core"
				pkg_suffix="fc$dist_version"
			else
				pkg_manager="yum"
				config_manager="yum-config-manager"
				enable_channel_flag="--enable"
				pre_reqs="yum-utils"
				pkg_suffix="el"
			fi
			(
				if ! is_dry_run; then
					set -x
				fi
				$sh_c "$pkg_manager install -y -q $pre_reqs"
				$sh_c "$config_manager --add-repo $yum_repo"

				if [ "$CHANNEL" != "stable" ]; then
					$sh_c "$config_manager $enable_channel_flag docker-ce-$CHANNEL"
				fi
				$sh_c "$pkg_manager makecache"
			)
			pkg_version=""
			if [ ! -z "$VERSION" ]; then
				if is_dry_run; then
					echo "# WARNING: VERSION pinning is not supported in DRY_RUN"
				else
					pkg_pattern="$(echo "$VERSION" | sed "s/-ce-/\\\\.ce.*/g" | sed "s/-/.*/g").*$pkg_suffix"
					search_command="$pkg_manager list --showduplicates 'docker-ce' | grep '$pkg_pattern' | tail -1 | awk '{print \$2}'"
					pkg_version="$($sh_c "$search_command")"
					echo "INFO: Searching repository for VERSION '$VERSION'"
					echo "INFO: $search_command"
					if [ -z "$pkg_version" ]; then
						echo
						echo "ERROR: '$VERSION' not found amongst $pkg_manager list results"
						echo
						exit 1
					fi
					# Cut out the epoch and prefix with a '-'
					pkg_version="-$(echo "$pkg_version" | cut -d':' -f 2)"
				fi
			fi
			(
				if ! is_dry_run; then
					set -x
				fi
				$sh_c "$pkg_manager install -y -q docker-ce$pkg_version"
			)
			echo_docker_as_nonroot
			exit 0
			;;
	esac
	exit 1
}

# wrapped up in a function so that we have some protection against only getting
# half the file during "curl | sh"
do_install
{% endcode %}
<!-- endtab -->
{% endtabs %}

## 制作镜像
```pwd /data/dockerfile/jenkins
[root@hdss7-200 jenkins]# ls -l
total 24
-rw------- 1 root root    98 Jan  17 15:58 config.json
-rw-r--r-- 1 root root   158 Jan  17 15:59 Dockerfile
-rwxr-xr-x 1 root root 13847 Jan  17 15:37 get-docker.sh
[root@hdss7-200 jenkins]# docker build . -t harbor.od.com/infra/jenkins:v2.164.1
[root@hdss7-200 jenkins]# docker push harbor.od.com/infra/jenkins:v2.164.1
```

## 准备jenkins的kubernetes资源配置清单
在运维主机`HDSS7-200.host.com`上的`/data/k8s-yaml`目录下
```pwd /data/k8s-yaml
[root@hdss7-200 k8s-yaml]# mkdir jenkins && cd jenkins
```
{% tabs jenkins-yaml%}
<!-- tab Deployment -->
[root@hdss7-200 k8s-yaml]# vi deployment.yaml
{% code %}
"kind": "Deployment",
"apiVersion": "extensions/v1beta1",
"metadata": {
  "name": "jenkins",
  "namespace": "infra",
  "labels": {
    "name": "jenkins"
  }
},
"spec": {
  "replicas": 1,
  "selector": {
    "matchLabels": {
      "name": "jenkins"
    }
  },
  "template": {
    "metadata": {
      "labels": {
        "app": "jenkins",
        "name": "jenkins"
      }
    },
    "spec": {
      "volumes": [
        {
          "name": "data",
          "hostPath": {
            "path": "/data/k8s-volume/jenkins_home",
            "type": ""
          }
        },
        {
          "name": "docker",
          "hostPath": {
            "path": "/run/docker.sock",
            "type": ""
          }
        }
      ],
      "containers": [
        {
          "name": "jenkins-test",
          "image": "harbor.od.com/infra/jenkins:v2.164.1",
          "ports": [
            {
              "containerPort": 8080,
              "protocol": "TCP"
            }
          ],
          "env": [
            {
              "name": "JAVA_OPTS",
              "value": "-Xmx128m -Xms128m"
            }
          ],
          "resources": {
            "limits": {
              "cpu": "300m",
              "memory": "128Mi"
            },
            "requests": {
              "cpu": "300m",
              "memory": "128Mi"
            }
          },
          "volumeMounts": [
            {
              "name": "data",
              "mountPath": "/var/jenkins_home"
            },
            {
              "name": "docker",
              "mountPath": "/run/docker.sock"
            }
          ],
          "terminationMessagePath": "/dev/termination-log",
          "terminationMessagePolicy": "File",
          "imagePullPolicy": "IfNotPresent"
        }
      ],
      "restartPolicy": "Always",
      "terminationGracePeriodSeconds": 30,
      "dnsPolicy": "Default",
      "nodeName": "10.4.7.21",
      "securityContext": {
        "runAsUser": 0
      },
      "schedulerName": "default-scheduler"
    }
  },
  "strategy": {
    "type": "RollingUpdate",
    "rollingUpdate": {
      "maxUnavailable": 1,
      "maxSurge": 1
    }
  },
  "revisionHistoryLimit": 7,
  "progressDeadlineSeconds": 600
}
{% endcode %}
<!-- endtab -->
<!-- tab Service-->
[root@hdss7-200 k8s-yaml]# vi svc.yaml
{% code %}
"kind": "Service",
"apiVersion": "v1",
"metadata": {
  "name": "jenkins",
  "namespace": "infra"
},
"spec": {
  "ports": [
    {
      "protocol": "TCP",
      "port": 8080,
      "targetPort": 8080
    }
  ],
  "selector": {
    "app": "jenkins"
  },
  "clusterIP": "None",
  "type": "ClusterIP",
  "sessionAffinity": "None"
}
{% endcode %}
<!-- endtab -->
<!-- tab Ingress-->
[root@hdss7-200 k8s-yaml]# vi ingress.yaml
{% code %}
"kind": "Ingress",
"apiVersion": "extensions/v1beta1",
"metadata": {
  "name": "jenkins",
  "namespace": "infra"
},
"spec": {
  "rules": [
    {
      "host": "jenkins.od.com",
      "http": {
        "paths": [
          {
            "backend": {
              "serviceName": "jenkins-test",
              "servicePort": 8080
            }
          }
        ]
      }
    }
  ]
}
{% endcode %}
<!-- endtab -->
{% endtabs %}