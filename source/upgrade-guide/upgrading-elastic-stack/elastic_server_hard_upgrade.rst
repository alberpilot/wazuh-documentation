.. Copyright (C) 2019 Wazuh, Inc.

.. _elastic_server_hard_upgrade:

Upgrading Elastic Stack from 6.x to 6.8
=======================================

Prepare the Elastic Stack
-------------------------

1. Stop the services:

  .. code-block:: console

    # systemctl stop logstash
    # systemctl stop filebeat
    # systemctl stop kibana

2. Add the new repository for Elastic Stack 7.x:

    * For CentOS/RHEL/Fedora:

      .. code-block:: console

        # rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch
        # cat > /etc/yum.repos.d/elastic.repo << EOF
        [elasticsearch-7.x]
        name=Elasticsearch repository for 7.x packages
        baseurl=https://artifacts.elastic.co/packages/7.x/yum
        gpgcheck=1
        gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
        enabled=1
        autorefresh=1
        type=rpm-md
        EOF

    * For Debian/Ubuntu:

      .. code-block:: console

        # curl -s https://artifacts.elastic.co/GPG-KEY-elasticsearch | apt-key add -
        # echo "deb https://artifacts.elastic.co/packages/7.x/apt stable main" | tee /etc/apt/sources.list.d/elastic-7.x.list

    * openSUSE:

      .. code-block:: console

        # rpm --import https://packages.elastic.co/GPG-KEY-elasticsearch
        # cat > /etc/zypp/repos.d/elastic.repo << EOF
        [elasticsearch-7.x]
        name=Elasticsearch repository for 7.x packages
        baseurl=https://artifacts.elastic.co/packages/7.x/yum
        gpgcheck=1
        gpgkey=https://artifacts.elastic.co/GPG-KEY-elasticsearch
        enabled=1
        autorefresh=1
        type=rpm-md
        EOF        

Upgrade Elasticsearch
---------------------

1. Disable shard allocation

  .. code-block:: bash

    curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
    {
      "persistent": {
        "cluster.routing.allocation.enable": "primaries"
      }
    }
    '

2. Stop non-essential indexing and perform a synced flush. (Optional)

  .. code-block:: bash

    curl -X POST "localhost:9200/_flush/synced"

3. Shut down a single node.

  .. code-block:: console

    # systemctl stop elasticsearch

4. Upgrade the node you shut down.

  * For CentOS/RHEL/Fedora:

    .. code-block:: console

      # yum install elasticsearch-6.8.6

  * For Debian/Ubuntu:

    .. code-block:: console

      # apt-get install elasticsearch=6.8.6
      # systemctl restart elasticsearch

5. Restart the service.

  .. code-block:: console

    # systemctl daemon-reload
    # systemctl restart elasticsearch

6. Start the newly-upgraded node and confirm that it joins the cluster by checking the log file or by submitting a *_cat/nodes* request:

  .. code-block:: bash

    curl -X GET "localhost:9200/_cat/nodes"

7. Reenable shard allocation.

  .. code-block:: bash

    curl -X PUT "localhost:9200/_cluster/settings" -H 'Content-Type: application/json' -d'
    {
      "persistent": {
        "cluster.routing.allocation.enable": null
      }
    }
    '

8. Before upgrading the next node, wait for the cluster to finish shard allocation.

  .. code-block:: bash

    curl -X GET "localhost:9200/_cat/health?v"

9. Repeat it for every Elasticsearch node.
10. Load the Wazuh template for Elasticsearch:

  .. code-block:: console

    # curl https://raw.githubusercontent.com/wazuh/wazuh/v3.11.4/extensions/elasticsearch/6.x/wazuh-template.json | curl -X PUT "http://localhost:9200/_template/wazuh" -H 'Content-Type: application/json' -d @-

Upgrade Logstash
^^^^^^^^^^^^^^^^

1. Upgrade the ``logstash`` package:

  a) For CentOS/RHEL/Fedora:

  .. code-block:: console

    # yum install logstash-6.8.6

  b) For Debian/Ubuntu:

  .. code-block:: console

    # apt-get install logstash=1:6.8.6-1

2. Download and set the Wazuh configuration for Logstash:

  a) Local configuration:

    .. code-block:: console

      # cp /etc/logstash/conf.d/01-wazuh.conf /backup_directory/01-wazuh.conf.bak
      # curl -so /etc/logstash/conf.d/01-wazuh.conf https://raw.githubusercontent.com/wazuh/wazuh/v3.11.4/extensions/logstash/6.x/01-wazuh-local.conf
      # usermod -a -G ossec logstash

  b) Remote configuration:

    .. code-block:: console

      # cp /etc/logstash/conf.d/01-wazuh.conf /backup_directory/01-wazuh.conf.bak
      # curl -so /etc/logstash/conf.d/01-wazuh.conf https://raw.githubusercontent.com/wazuh/wazuh/v3.11.4/extensions/logstash/6.x/01-wazuh-remote.conf

3. Start the Logstash service:

  .. code-block:: console

    # systemctl daemon-reload
    # systemctl start logstash.service

Upgrade Filebeat
----------------

1. Upgrade Filebeat.

  * For CentOS/RHEL/Fedora:

    .. code-block:: console

      # yum install filebeat-6.8.6

  * For Debian/Ubuntu:

    .. code-block:: console

      # apt-get install filebeat=6.8.6

2. Update the configuration file.

  .. code-block:: console

    # cp /etc/filebeat/filebeat.yml /backup/filebeat.yml.backup
    # curl -so /etc/filebeat/filebeat.yml https://raw.githubusercontent.com/wazuh/wazuh/v3.11.4/extensions/filebeat/6.x/filebeat.yml
    # chmod go+r /etc/filebeat/filebeat.yml

3. Restart Filebeat.

  .. code-block:: console

    # systemctl daemon-reload
    # systemctl restart filebeat

Upgrade Kibana
--------------

Upgrade Kibana
^^^^^^^^^^^^^^

1. Upgrade the ``kibana`` package:

  a) For CentOS/RHEL/Fedora:

  .. code-block:: console

    # yum install kibana-6.8.6

  b) For Debian/Ubuntu:

  .. code-block:: console

    # apt-get install kibana=6.8.6

2. Uninstall the Wazuh app from Kibana:

  a) Update file permissions. This will avoid several errors prior to updating the app:

  .. code-block:: console

    # chown -R kibana:kibana /usr/share/kibana/optimize
    # chown -R kibana:kibana /usr/share/kibana/plugins

  b) Remove the Wazuh app:

  .. code-block:: console

    # cd /usr/share/kibana/
    # sudo -u kibana bin/kibana-plugin remove wazuh

3. Upgrade the Wazuh app:

  .. code-block:: console

    # cd /usr/share/kibana/
    # rm -rf optimize/bundles
    # sudo -u kibana NODE_OPTIONS="--max-old-space-size=3072" bin/kibana-plugin install https://packages.wazuh.com/wazuhapp/wazuhapp-3.11.4_6.8.6.zip

  .. warning::

    The Wazuh app installation process may take several minutes. Please wait patiently.

4. Start the Kibana service:

  .. code-block:: console

    # systemctl daemon-reload
    # systemctl enable kibana.service
    # systemctl start kibana.service

Disabling repositories
^^^^^^^^^^^^^^^^^^^^^^

    * For CentOS/RHEL/Fedora:

      .. code-block:: console

        # sed -i "s/^enabled=1/enabled=0/" /etc/yum.repos.d/elastic.repo

    * For Debian/Ubuntu:

      .. code-block:: console

        # sed -i "s/^deb/#deb/" /etc/apt/sources.list.d/elastic-7.x.list
        # apt-get update

      Alternatively, you can set the package state to ``hold``, which will stop updates (although you can still upgrade it manually using ``apt-get install``).

      .. code-block:: console

        # echo "elasticsearch hold" | sudo dpkg --set-selections
        # echo "kibana hold" | sudo dpkg --set-selections

    * For openSUSE:

      .. code-block:: console

        # sed -i "s/^enabled=1/enabled=0/" /etc/zypp/repos.d/elastic.repo