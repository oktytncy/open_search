### Create a Virtual Environment [Optional]

Some environments can be managed externally by a system package manager, such as Homebrew in macOS. 

This means that your system may prevent you from installing packages globally to avoid conflicts with packages managed by the system package manager. This may pose an obstacle to package installation.

**To avoid this issue, you can Create a Virtual Environment before running python or pip commands.**

```bash
cd "<YOUR-INSTALLATION-PATH>"
```

#### on Linux/Mac/..

Install environment and activate the virtual environment.

```python
python3.11 -m venv myenv
```

```bash
source myenv/bin/activate
```

## Set Up Development OpenSearch Cluster

1. Create a directory for your OpenSearch cluster:

    ```shell
    mkdir opensearch-cluster
    cd opensearch-cluster
    ```

2. Create the Docker Compose file

    > In this directory, create `docker-compose.yml` and paste the development compose contents into it:
