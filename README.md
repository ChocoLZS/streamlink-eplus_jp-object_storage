# Streamlink, eplus.jp and Object Storage

> DMCA is coming...?

## What does this docker image do

- Download the live streaming or VOD from [eplus - Japan most famous ticket play-guide](https://ib.eplus.jp/) via Streamlink.
- Transcode the downloaded TS file to MP4 via FFmpeg.
- Upload both the TS and MP4 files to Azure Storage container via Azure CLI.

## Details

### Storage requirement

For lives of Liella, the size of a 4-hour MPEG-TS record with the best quality is about 9.88 GB (9.2 GiB). Since we may transcode MPEG-TS to MP4 (a bit smaller) and keep both files, 20 GB free disk space is required.

### Prepare your object storage

#### AWS S3-compatible preparation (simpler)

Create your own object storage:

- AWS:
  - [Creating, configuring, and working with Amazon S3 buckets - Amazon Simple Storage Service](https://docs.aws.amazon.com/AmazonS3/latest/userguide/creating-buckets-s3.html)
- DigitalOcean:
  - [How to Create Spaces :: DigitalOcean Documentation](https://docs.digitalocean.com/products/spaces/how-to/create/)
  - [Setting Up s3cmd 2.x with DigitalOcean Spaces :: DigitalOcean Documentation](https://docs.digitalocean.com/products/spaces/resources/s3cmd/)
- Linode:
  - [Object Storage - Get Started | Linode](https://www.linode.com/docs/products/storage/object-storage/get-started/)
  - [Deploy a Static Site using Hugo and Object Storage | Linode](https://www.linode.com/docs/guides/host-static-site-object-storage/)
- Vultr:
  - [Vultr Object Storage - Vultr.com](https://www.vultr.com/docs/vultr-object-storage)

Environment variables:

| Name | Description
| - | -
| S3_BUCKET | URL in `s3://bucket-name/dir-name/` style
| S3_HOSTNAME | `s3-eu-west-1.amazonaws.com` , `nyc3.digitaloceanspaces.com` , `us-east-1.linodeobjects.com` or `ewr1.vultrobjects.com` , for example
| AWS_ACCESS_KEY_ID | The access key
| AWS_SECRET_ACCESS_KEY | The secret key

#### Azure preparation

Create a service principal on Azure.

- [Create an Azure AD app and service principal in the portal - Microsoft identity platform | Microsoft Docs](https://docs.microsoft.com/en-us/azure/active-directory/develop/howto-create-service-principal-portal)
- [Create an Azure service principal – Azure CLI | Microsoft Docs](https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli)

For [`azure-cli`](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) :

```console
$ az login --use-device-code
$ az ad sp create-for-rbac --role 'Contributor' --name "${name}" --scopes "/subscriptions/${subscription}/resourceGroups/${resourceGroup}/providers/Microsoft.Storage/storageAccounts/${AZURE_STORAGE_ACCOUNT}"

```

Environment variables:

| Name | Description
| - | -
| AZ_SP_APPID | Application (client) ID
| AZ_SP_PASSWORD | Client secret
| AZ_SP_TENANT | Directory (tenant) ID
| AZURE_STORAGE_ACCOUNT | Azure storage account
| AZ_STORAGE_CONTAINER_NAME | Storage container name

### Launch the container

#### Docker

Install [Docker Engine](https://docs.docker.com/engine/). I recommend that you use [Compose V2](https://docs.docker.com/compose/#compose-v2-and-the-new-docker-compose-command) by executing the new [`docker compose`](https://docs.docker.com/engine/reference/commandline/compose/) command.

Create a service:

```YAML
# docker-compose.yml

services:
  sl:
    image: docker.io/pzhlkj6612/streamlink-eplus_jp-object_storage
    volumes:
      - ./SL-downloads:/SL-downloads:rw
    environment:
      # base file name (required)
      - OUTPUT_FILENAME_BASE=

      # switches
      - NO_DOWNLOAD_STREAM=  # disable streamlink, generate a still image video.
      - NO_TRANSCODE=        # disable FFmpeg transcode, generate a dummy video.
      - NO_S3=               # disable s3cmd.
      - NO_AZURE=            # disable azure-cli.

      # streamlink
      - EPLUS_JP_STREAM_URL=
      - EPLUS_JP_STREAM_QUALITY=

      # s3cmd
      - AWS_ACCESS_KEY_ID=
      - AWS_SECRET_ACCESS_KEY=
      - S3_BUCKET=
      - S3_HOSTNAME=

      # azure-cli
      - AZURE_STORAGE_ACCOUNT=
      - AZ_SP_APPID=
      - AZ_SP_PASSWORD=
      - AZ_SP_TENANT=
      - AZ_STORAGE_CONTAINER_NAME=

```

Run it:

```console
$ docker compose up sl

```

For developers who want to build the image themselves:

```console
$ DOCKER_BUILDKIT=1 docker build --tag ${tag} .
  ^^^^^^^^^^^^^^^^^
     !important!

```

#### Podman

[Install Podman](https://podman.io/getting-started/installation). Create a pod and a "[hostPath](https://kubernetes.io/docs/concepts/storage/volumes/#hostpath)" volume:

```YAML
# pod.yaml

apiVersion: v1
kind: Pod
metadata:
  name: sl
spec:
  volumes:
    - name: SL-downloads
      hostPath:
        path: ./SL-downloads
        type: Directory
  restartPolicy: Never
  containers:
    - name: sl
      image: docker.io/pzhlkj6612/streamlink-eplus_jp-object_storage
      resources: {}
      volumeMounts:
        - mountPath: /SL-downloads
          name: SL-downloads
      env:
        # base file name (required)
        - name: OUTPUT_FILENAME_BASE
          value: ""

        # switches
        - name: NO_DOWNLOAD_STREAM  # disable streamlink, generate a still image video.
          value: ""
        - name: NO_TRANSCODE        # disable FFmpeg transcode, generate a dummy video.
          value: ""
        - name: NO_S3               # disable s3cmd.
          value: ""
        - name: NO_AZURE            # disable azure-cli.
          value: ""

        # streamlink
        - name: EPLUS_JP_STREAM_URL
          value: ""
        - name: EPLUS_JP_STREAM_QUALITY
          value: ""

        # s3cmd
        - name: AWS_ACCESS_KEY_ID
          value: ""
        - name: AWS_SECRET_ACCESS_KEY
          value: ""
        - name: S3_BUCKET
          value: ""
        - name: S3_HOSTNAME
          value: ""

        # azure-cli
        - name: AZURE_STORAGE_ACCOUNT
          value: ""
        - name: AZ_SP_APPID
          value: ""
        - name: AZ_SP_PASSWORD
          value: ""
        - name: AZ_SP_TENANT
          value: ""
        - name: AZ_STORAGE_CONTAINER_NAME
          value: ""

```

Finally, play it:

```console
$ podman play kube ./pod.yaml  # 開演！

```

For developers who want to build the image themselves:

```console
$ podman build --tag ${tag} .

```

## Credits

- Container Technologies:
  - Open Container Initiative (OCI).
  - Docker and Docker Compose.
  - Podman and Kubernetes.
- Useful open-source programs and tools:
  - [s3tools/s3cmd](https://github.com/s3tools/s3cmd).
  - [Azure/azure-cli](https://github.com/Azure/azure-cli).
  - [streamlink/streamlink](https://github.com/streamlink/streamlink) and [pmrowla/streamlink-plugins](https://github.com/pmrowla/streamlink-plugins).
  - [BtbN/FFmpeg-Builds](https://github.com/BtbN/FFmpeg-Builds).
  - I'm using [shell-format - Visual Studio Marketplace](https://marketplace.visualstudio.com/items?itemName=foxundermoon.shell-format) to format my bash shell script.
- Platforms:
  - AWS, DigitalOcean, Linode and Vultr.
  - Microsoft Azure.
  - [Stack Exchange](https://stackexchange.com/) website group.
  - Linux and Ubuntu.
