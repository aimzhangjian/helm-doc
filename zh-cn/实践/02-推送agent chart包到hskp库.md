# 推送agent chart包到hskp库

### 添加仓库 
```
helm repo add hskp --username hsopDeployer --password kjDJHGDJshfaflasfa34 http://harbor.open.hand-china.com/chartrepo/hskp

helm repo add choerodon https://chart.choerodon.com.cn/hzero/hrds
```
### 拉取chart包
```
helm pull choerodon/hskp-devops-cluster-agent --version 0.2.0
```
### 推送chart包
```
helm push hskp-devops-cluster-agent-0.1.0.tgz --username hsopDeployer --password kjDJHGDJshfaflasfa34 http://harbor.open.hand-china.com/chartrepo/hskp
```