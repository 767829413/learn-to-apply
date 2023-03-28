# kubectl命令行管理工具

1. `基础命令: 创建资源、更新资源、删除`
2. `部署命令: 部署状态、发布记录、回滚，扩容/缩容`
3. `集群管理命令: 查看资源利用、节点管理`

```bash
kubectl exec -it nginx-f89759699-xhxhi bash
```

4. `故障诊断和调试命令:查看资源信息、查看容器日志、进入容器、拷贝、端口映射`
5. `高级命令:部署资源、更新资源`
6. `设置命令: 资源类型相关信息查看,命令补全`

```bash
yum install -y bash-completion && source /usr/share/bash-completion/bash_completion && source <(kubectl completion bash)
```

7. `通用的命令选项`

```bash
# 尝试跑下资源,但不具体执行
--dry-run
# 输出的格式 例如 wide，yaml，json
-o，--output=
```

kubectl get 通用的选项:

```bash
# 输出的格式
-o，--output=
# 所有命名空间
-A，--all-namespaces=
# 排序
--sort-by= 
# 查看标签
--show-labels
# 根据标签查询资源 -l app=naginx
-l,--selector
```

`--grace-period=0 --force 强制删除资源`

<table>
    <tr>
        <th>类型</th><th>命令</th><th>描叙</th>
    </tr>
    <tr>
        <td rowspan="8">基础命令</td>
        <td>create</td>
        <td>通过文件名或标准输入创建资源</td>
    </tr>
    <tr>
        <td>expose</td>
        <td>为Deployment, Pod 创建 Service</td>
    </tr>
    <tr>
        <td>run</td>
        <td>在集群中运行一个特定的镜像</td>
    </tr>
    <tr>
        <td>set</td>
        <td>在对象上设置特定的功能</td>
    </tr>
    <tr>
        <td>explain</td>
        <td>文档参考资料</td>
    </tr>
    <tr>
        <td>get</td>
        <td>显示一个或多个资源</td>
    </tr>
    <tr>
        <td>edit</td>
        <td>使用系统编辑器编辑一个资源</td>
    </tr>
    <tr>
        <td>delete</td>
        <td>通过文件名,标准输入,资源名称或标签选择器来删除资源</td>
    </tr>
    <tr>
        <td rowspan="4">部署命令</td>
        <td>rollout</td>
        <td>管理Deployment,Daemonset资源的发布(状态,发布记录,回滚等)</td>
    </tr>
    <tr>
        <td>rolling-update</td>
        <td>滚动升级,适用ReplicationController</td>
    </tr>
    <tr>
        <td>scale</td>
        <td>对Deployment, ReplicaSet, RC或Job资源扩容或缩容Pod数量</td>
    </tr>
    <tr>
        <td>autoscale</td>
        <td>为Deploy, RS, RC配置自动伸缩规则(依赖metrics-server和hpa)</td>
    </tr>
    <tr>
        <td rowspan="7">集群管理命令</td>
        <td>certificate</td>
        <td>修改证书资源</td>
    </tr>
    <tr>
        <td>cluster-info</td>
        <td>显示集群信息</td>
    </tr>
    <tr>
        <td>top</td>
        <td>查看资源利用率(依赖metrics-server)</td>
    </tr>
    <tr>
        <td>cordon</td>
        <td>标记节点不可调度</td>
    </tr>
    <tr>
        <td>uncordon</td>
        <td>标记节点可调度</td>
    </tr>
    <tr>
        <td>drain</td>
        <td>驱逐节点上的应用，准备下线维护</td>
    </tr>
    <tr>
        <td>taint</td>
        <td>修改节点taint标记</td>
    </tr>
    <tr>
        <td rowspan="7">故障诊断和调试命令</td>
        <td>describe</td>
        <td>显示资源详细信息</td>
    </tr>
    <tr>
        <td>logs</td>
        <td>查看Pod内容器日志，如果Pod有多个容器，-c参数指定容器名称</td>
    </tr>
    <tr>
        <td>attach</td>
        <td>附加到Pod内的一个容器</td>
    </tr>
    <tr>
        <td>exec</td>
        <td>在容器内执行命令</td>
    </tr>
    <tr>
        <td>port-forward</td>
        <td>为Pod创建本地端口映射</td>
    </tr>
    <tr>
        <td>proxy</td>
        <td>为Kubernetes API server创建代理</td>
    </tr>
    <tr>
        <td>cp</td>
        <td>拷贝文件或目录到容器中，或者从容器内向外拷贝</td>
    </tr>
    <tr>
        <td rowspan="4">高级命令</td>
        <td>apply</td>
        <td>从文件名或标准输入对资源创建/更新</td>
    </tr>
    <tr>
        <td>patch</td>
        <td>使用补丁方式修改、更新资源的某些字段</td>
    </tr>
    <tr>
        <td>replace</td>
        <td>从文件名或标准输入替换一个资源</td>
    </tr>
    <tr>
        <td>convert</td>
        <td>在不同API版本之间转换对象定义</td>
    </tr>
    <tr>
        <td rowspan="4">设置命令</td>
        <td>label</td>
        <td>给资源设置、更新标签</td>
    </tr>
    <tr>
        <td>annotate</td>
        <td>给资源设置、更新注解</td>
    </tr>
    <tr>
        <td>completion</td>
        <td>kubectl工具自动补全 yum install -y bash-completion && source /usr/share/bash-completion/bash_completion && source <(kubectl completion bash) (依赖软件包 bash-completion)</td>
    </tr>
    <tr>
        <td>api-resources</td>
        <td>查看所有支持的资源</td>
    </tr>
    <tr>
        <td rowspan="4">其他命令</td>
        <td>api-versions</td>
        <td>打印受支持的API版本</td>
    </tr>
    <tr>
        <td>config</td>
        <td>修改kubeconfig文件(用于访问API，比如配置认证信息)</td>
    </tr>
    <tr>
        <td>help</td>
        <td>所有命令帮助</td>
    </tr>
    <tr>
        <td>version</td>
        <td>查看kubectl和k8s版本</td>
    </tr>
</table>