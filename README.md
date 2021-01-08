## Installing Opentaxii Server in Docker Container
```
apt-get update
apt-get -y install python3 python3-pip curl git default-libmysqlclient-dev
pip3 install opentaxii
pip3 install mysqlclient
pip3 install gunicorn
```
```
cat > config.yaml <<EOF
domain: localhost:9000
support_basic_auth: yes

persistence_api:
  class: opentaxii.persistence.sqldb.SQLDatabaseAPI
  parameters:
    db_connection: mysql://taxii:misp_taxii@misp_db/taxiipersist
    create_tables: yes

auth_api:
  class: opentaxii.auth.sqldb.SQLDatabaseAPI
  parameters:
    db_connection: mysql://taxii:misp_taxii@misp_db/taxiiauth
    create_tables: yes
    secret: 123@qwe

logging:
  opentaxii: info
  root: info

hooks: misp_taxii_hooks.hooks
# Sample configuration for misp_taxii_server

zmq:
    host: localhost
    port: 50000

misp:
    url: http://172.18.0.3
    api: vguhGDqNcC32ICeEm0kwQncbkBrzn8iyBZGt1dp5

taxii:
    auth:
        username: ABC
        password: CDE
    collections:
        - collection
EOF
```
```
export OPENTAXII_CONFIG=config.yaml
```
```
cat > data.yaml << EOF
services:
  - id: inbox_a
    type: inbox
    address: /services/inbox-a
    description: Custom Inbox Service Description A
    destination_collection_required: yes
    accept_all_content: yes
    authentication_required: no
    protocol_bindings:
      - urn:taxii.mitre.org:protocol:http:1.0

  - id: inbox_b
    type: inbox
    address: /services/inbox-b
    description: Custom Inbox Service Description B
    destination_collection_required: yes
    accept_all_content: no
    authentication_required: yes
    supported_content:
      - urn:stix.mitre.org:xml:1.1.1
      - urn:custom.example.com:json:0.0.1
    protocol_bindings:
      - urn:taxii.mitre.org:protocol:http:1.0

  - id: discovery_a
    type: discovery
    address: /services/discovery-a
    description: Custom Discovery Service description
    advertised_services:
      - inbox_a
      - inbox_b
      - discovery_a
      - collection_management_a
      - poll_a
    protocol_bindings:
      - urn:taxii.mitre.org:protocol:http:1.0
      - urn:taxii.mitre.org:protocol:https:1.0

  - id: collection_management_a
    type: collection_management
    address: /services/collection-management-a
    description: Custom Collection Management Service description
    protocol_bindings:
      - urn:taxii.mitre.org:protocol:http:1.0
      - urn:taxii.mitre.org:protocol:https:1.0

  - id: poll_a
    type: poll
    address: /services/poll-a
    description: Custom Poll Service description
    subscription_required: no
    max_result_count: 100
    max_result_size: 10
    authentication_required: yes
    protocol_bindings:
      - urn:taxii.mitre.org:protocol:http:1.0

collections:
  - name: collection-a
    available: true
    accept_all_content: true
    type: DATA_SET
    service_ids:
      - inbox_a
      - collection_management_a
      - poll_a

  - name: collection-b
    available: true
    accept_all_content: false
    supported_content:
      - urn:stix.mitre.org:xml:1.1.1
    service_ids:
      - inbox_a
      - inbox_b
      - collection_management_a
      - poll_a

  - name: collection-c
    available: true
    accept_all_content: false
    supported_content:
      - urn:stix.mitre.org:xml:1.1.1
      - urn:custom.bindings.com:json:0.0.1
    service_ids:
      - inbox_a
      - collection_management_a
      - poll_a

  - name: col-not-available
    available: false
    service_ids:
      - inbox_b
      - collection_management_a

accounts:
  - username: test
    password: test
    permissions:
      collection-a: modify
      collection-b: read
      collection-c: read
      collection-xyz: some
  - username: admin
    password: admin
    is_admin: yes
EOF
```
```
git clone --recursive https://github.com/MISP/MISP-Taxii-Server.git
git clone --recursive https://github.com/FloatingGhost/MISP-Taxii-Server
cd MISP-Taxii-Server/
python3 setup.py install
pip3 install -r REQUIREMENTS.txt
cd ..
opentaxii-sync-data data.yaml
```
```
#opentaxii-run-dev
```
```
gunicorn opentaxii.http:app --bind localhost:9000 --config python:opentaxii.http
```
```
curl -d 'username=guest&password=guest' http://localhost:9000/management/auth
{
  "token": "eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJhY2NvdW50X2lkIjoxLCJleHAiOjE2MDg0MzkyNTl9.g2mkWIkreaQDm_eAMeE-_eGA3EbzBOjJCRlXW6HXebQ"
}
```
```
taxii-poll \
            --path http://localhost:9000/services/poll \
            --collection my_collection \
            --header Authorization:'Bearer eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJhY2NvdW50X2lkIjoxLCJleHAiOjE2MDg0MzkyNTl9.g2mkWIkreaQDm_eAMeE-_eGA3EbzBOjJCRlXW6HXebQ'
```
```
taxii-discovery --path http://localhost:9000/services/discovery

taxii-collections --path http://localhost:9000/services/collection-management

taxii-push --path http://localhost:9000/services/inbox -f /MISP-Taxii-Server/tests/sample.xml --dest my_collection --username admin --password admin

taxii-poll \
            --host hailataxii.com \
            --port 80 \
            --path /taxii-data \
            --collection guest.Abuse_ch \
            --username guest \
            --password guest \
            --jwt-auth /taxii-data

taxii-discovery --path https://taxii.trustar.co/services/discovery
taxii-collections --path https://taxii.trustar.co/services/collection-management
taxii-poll --path https://taxii.trustar.co/services/poll --collection collection-indicator-SOFTWARE
```
