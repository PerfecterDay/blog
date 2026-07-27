# oci 镜像的格式
{docsify-updated}

在 Orbstack 中执行 `mac docker save busybox:latest -o busybox.tar` ，可以导出 busybox 镜像， `tar -xvf busybox.tar -C busybox/` 解压后可以看到如下的目录结构：
```
├── blobs
│   └── sha256
│       ├── 66cb17eae60e0bf660a4cf7c5fada7a748febe4c92dd972f1735af7a7c7c740d
│       ├── d36542e65e5233c818056590c74971a4841545a3255510ad7949fc0b3b26a64a
│       ├── e01d3e92a09ceeed87b7c96483cbd58cc22add26e191d981f023652d485ad74f
│       └── e0e8b3cbfed68a90084781e2962f9c0deead51c5a3f11a488eef0283a4284bc2
├── index.json
├── manifest.json
├── oci-layout
└── repositories
```


## 从0开始纯手工构造 image
1. 创建工作目录： `mkdir handmade-image`
2. 创建 rootfs 目录： `mkdir rootfs`
3. 创建 hello.sh 脚本： 
```
cat > rootfs/hello.sh <<EOF
#!/bin/sh
echo "hello from my handmade docker image"
EOF
```
4. 打包 rootfs：`tar -C rootfs -cf layer.tar .`
5. 计算 layer digest: `sha256sum layer.tar`，假设为 `64ac98a1b45aa07bff6e50f2d65583cc0e95bf2741a2813de964bd2d810bbf1d`
6. 创建 OCI blobs: `mkdir -p blobs/sha256`
7. 将 layer 放到 blobs 目录中：`cp layer.tar blobs/sha256/64ac98a1b45aa07bff6e50f2d65583cc0e95bf2741a2813de964bd2d810bbf1d`
8. `stat -c%s layer.tar` 查看 layer 的 size，假设为 10240
9. 创建 image config，注意  rootfs.diff_ids 中的 sha256 值为 layer 的 digest 值,第5步计算出来的:
```
cat > config.json <<EOF
{
  "architecture":"arm64",
  "os":"linux",

  "config":{
    "Cmd":[
      "/hello.sh"
    ]
  },

  "rootfs":{
    "type":"layers",
    "diff_ids":[
      "sha256:64ac98a1b45aa07bff6e50f2d65583cc0e95bf2741a2813de964bd2d810bbf1d"
    ]
  }
}
EOF
```
9. config 也要进入 blobs： `sha256sum config.json` ，假设结果为 eb2b5a6d83bd03cf5f50cfda4bb55a4dfcc1209a93b653b195076a6a43d1417a ， 再执行 `cp config.json blobs/sha256/eb2b5a6d83bd03cf5f50cfda4bb55a4dfcc1209a93b653b195076a6a43d1417a`
10. 创建 manifest
```
cat > manifest.json <<EOF
{
 "schemaVersion":2,

 "config":{
   "mediaType":
   "application/vnd.oci.image.config.v1+json",

   "digest":
   "sha256:eb2b5a6d83bd03cf5f50cfda4bb55a4dfcc1209a93b653b195076a6a43d1417a",

   "size":243
 },


 "layers":[
 {
   "mediaType":
   "application/vnd.oci.image.layer.v1.tar",

   "digest":
   "sha256:64ac98a1b45aa07bff6e50f2d65583cc0e95bf2741a2813de964bd2d810bbf1d",

   "size":10240
 }
 ]
}
EOF
```
11. manifest 放入 blobs: `sha256sum manifest.json` ，假设结果为 619f7d0a66704d45f9eee454ec3ca8131b83bf7bc1673a31d1da448a86ce184b ， 再执行 `cp manifest.json blobs/sha256/619f7d0a66704d45f9eee454ec3ca8131b83bf7bc1673a31d1da448a86ce184b`
12. 创建 oci-layout
```
cat > oci-layout <<EOF
{
 "imageLayoutVersion":"1.0.0"
}
EOF
```
13. 创建 index.json
```
cat > index.json <<EOF
{
 "schemaVersion":2,

 "manifests":[
 {
   "mediaType":
   "application/vnd.oci.image.manifest.v1+json",

   "digest":
   "sha256:619f7d0a66704d45f9eee454ec3ca8131b83bf7bc1673a31d1da448a86ce184b",

   "size":404,

   "annotations":{
     "org.opencontainers.image.ref.name":
     "manual-nginx:1.0"
   }
 }
 ]
}
EOF
```
14. 打包： `tar -cf manual-nginx.tar handmade-image`
15. 导入 docker: `docker load < manual-nginx.tar`
16. 运行 `docker run manual-nginx:1.0`


`skopeo inspect oci:./handmade-image`