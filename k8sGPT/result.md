# k8sGPT
## 検証環境
* kubernetes 1.31
* RHEL 9.5

## インストール
```
$ curl -LO https://github.com/k8sgpt-ai/k8sgpt/releases/download/v0.3.24/k8sgpt_amd64.rpm
$ sudo rpm -ivh -i k8sgpt_amd64.rpm
$ k8sgpt version
k8sgpt: 0.3.24 (eac9f07), built at: unknown
```
* 参考
  * https://github.com/k8sgpt-ai/docs/blob/main/docs/getting-started/installation.md

## 動作確認
* OpenAIのAPI Keyを登録をする
```
$ k8sgpt auth add --backend openai --model gpt-3.5-turbo
Enter openai Key: openai added to the AI backend provider list
```
* imageが存在しないPodを起動する
```
$ k create -f broken-pod.yaml
pod/broken-nginx created
```
* kubectlで状態確認
```
$ k get pods
NAME           READY   STATUS             RESTARTS   AGE
broken-nginx   0/1     ImagePullBackOff   0          20s
```
* k8sGPTで問題の原因を確認する
```
$ k8sgpt analyze
AI Provider: openai

0 default/broken-nginx(broken-nginx)
- Error: Back-off pulling image "nginx:brokenxxxx"
```
このとき、問題のあるPodを表示しているだけでLLMは使用していない

* `--explain` を付加して、詳しく見てみる
```
$ k8sgpt analyze --explain
 100% |████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| (1/1, 44 it/min)        
AI Provider: openai

0 default/broken-nginx(broken-nginx)
- Error: Back-off pulling image "nginx:brokenxxxx"
Error: The Kubernetes is unable to pull the image "nginx:brokenxxxx" due to repeated failures.

Solution: 
1. Check the image name and tag for typos.
2. Ensure the image repository is accessible.
3. Verify network connectivity.
4. Retry pulling the image after fixing any issues.
```

## 動作確認(Javaアプリケーション)
* JavaアプリケーションのJavaオプションが誤っている場合を想定
* サンプルアプリケーション teastore を実行する
```
$ git clone https://github.com/DescartesResearch/TeaStore.git
$ k create -f TeaStore/examples/kubernetes/teastore-clusterip.yaml
$ k get pod
NAME                                    READY   STATUS             RESTARTS   AGE
teastore-auth-7985f5bcdd-62ss4          1/1     Running            0          58s
teastore-db-7fb64588cb-m7wlr            1/1     Running            0          58s
teastore-image-cdbdc6546-n8lbr          1/1     Running            0          58s
teastore-persistence-55d4f8c565-vs9zp   1/1     Running            0          58s
teastore-recommender-665898fcf4-hdmwh   1/1     Running            0          58s
teastore-registry-5bc46589c5-k6g5k      1/1     Running            0          58s
teastore-webui-65ddf7fdf6-nvgbc         1/1     Running            0          58s
```
* TeaStore/examples/kubernetes/teastore-clusterip.yaml のwebuiのdeploymentの環境変数に存在しないJavaオプションを追記する
```
            - name: JAVA_TOOL_OPTIONS
              value: "-Xmx300m -XX:MaxMetaspaceSize=256m -XX:BrokenOptions=XXX"
```

* 再度実行
```
$ k delete -f TeaStore/examples/kubernetes/teastore-clusterip.yaml
$ k create -f TeaStore/examples/kubernetes/teastore-clusterip.yaml
$ $ k get pod
NAME                                    READY   STATUS             RESTARTS   AGE
teastore-auth-7985f5bcdd-d89vs          1/1     Running            0          12s
teastore-db-7fb64588cb-8cgqp            1/1     Running            0          12s
teastore-image-cdbdc6546-2wbjg          1/1     Running            0          12s
teastore-persistence-55d4f8c565-4s9rs   1/1     Running            0          12s
teastore-recommender-665898fcf4-8mfpg   1/1     Running            0          12s
teastore-registry-5bc46589c5-9hwzl      1/1     Running            0          12s
teastore-webui-7f75775dbd-bdhwh         0/1     Error              0          12s
```
webuiでエラーが発生している

* k8sGPTでみてみる
```
$ k8sgpt analyze
AI Provider: openai

0 default/teastore-webui(teastore-webui)
- Error: Service has not ready endpoints, pods: [Pod/teastore-webui-7f75775dbd-bdhwh], expected 1

1 default/teastore-webui-7f75775dbd-bdhwh(Deployment/teastore-webui)
- Error: back-off 1m20s restarting failed container=teastore-webui pod=teastore-webui-7f75775dbd-bdhwh_default(d672f48d-6001-4e04-9a06-4e451fea3a0f)
- Error: the last termination reason is Error container=teastore-webui pod=teastore-webui-7f75775dbd-bdhwh

k8sgpt analyze --explain
 100% |████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████████| (2/2, 44 it/min)        
AI Provider: openai

0 default/teastore-webui(teastore-webui)
- Error: Service has not ready endpoints, pods: [Pod/teastore-webui-7f75775dbd-bdhwh], expected 1
Error: Service has no ready endpoints due to the pod teastore-webui-7f75775dbd-bdhwh not being ready.

Solution: 
1. Check the status of the pod teastore-webui-7f75775dbd-bdhwh.
2. Ensure the pod is running and ready.
3. Restart the pod if necessary.
4. Monitor the service for any changes in the endpoints.
1 default/teastore-webui-7f75775dbd-bdhwh(Deployment/teastore-webui)
- Error: back-off 1m20s restarting failed container=teastore-webui pod=teastore-webui-7f75775dbd-bdhwh_default(d672f48d-6001-4e04-9a06-4e451fea3a0f)
- Error: the last termination reason is Error container=teastore-webui pod=teastore-webui-7f75775dbd-bdhwh
Error: Container teastore-webui in pod teastore-webui-7f75775dbd-bdhwh failed to start and keeps restarting.
Solution: 1. Check the logs of the container for more specific error messages. 2. Verify the configuration and dependencies of the container. 3. Restart the pod to see if the issue resolves.
```
エラー文以上の情報はないかな...