# SonarQube Image 

This implementation use SonarQube lts community image from docker.io

### sonarqube

# postgres pod running on OCP crashing frequently with CrashLoopBackOff events
https://access.redhat.com/solutions/5668581

# reference
https://github.com/redhat-cop/containers-quickstarts/blob/master/sonarqube/.openshift/templates/sonarqube-deployment-template.yml

# create project
oc new-project sonarqube

# set configmap
oc create configmap sonar-conf --from-file=./sonar.properties

or 

cat << 'EOF' | oc create -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: sonar-conf
  namespace: sonarqube
data:
  sonar.properties: |-
    sonar.log.console=true
    sonar.jdbc.username=${env:JDBC_USERNAME}
    sonar.jdbc.password=${env:JDBC_PASSWORD}
    sonar.jdbc.url=${env:JDBC_URL}
    sonar.security.realm=${env:LDAP_REALM}
    sonar.forceAuthentication=${env:FORCE_AUTHENTICATION}
    sonar.authenticator.createUsers=${env:SONAR_AUTOCREATE_USERS}
    http.proxyHost=${env:PROXY_HOST}
    http.proxyPort=${env:PROXY_PORT}
    http.proxyUser=${env:PROXY_USER}
    http.proxyPassword=${env:PROXY_PASSWORD}
    ldap.bindDn=${env:LDAP_BINDDN}
    ldap.bindPassword=${env:LDAP_BINDPASSWD}
    ldap.url=${env:LDAP_URL}
    ldap.contextFactoryClass=${env:LDAP_CONTEXTFACTORY}
    ldap.StartTLS=${env:LDAP_STARTTLS}
    ldap.authentication=${env:LDAP_AUTHENTICATION}
    ldap.user.baseDn=${env:LDAP_USER_BASEDN}
    ldap.user.request=${env:LDAP_USER_REQUEST}
    ldap.user.realNameAttribute=${env:LDAP_USER_REAL_NAME_ATTR}
    ldap.user.emailAttribute=${env:LDAP_USER_EMAIL_ATTR}
    ldap.group.baseDn=${env:LDAP_GROUP_BASEDN}
    ldap.group.request=${env:LDAP_GROUP_REQUEST}
    ldap.group.idAttribute=${env:LDAP_GROUP_ID_ATTR}
EOF

# import sonarqube container from docker.io
oc import-image sonarqube:latest --from=docker.io/library/sonarqube:lts-community --confirm --force -n sonarqube

# create persistent volumes
cat <<EOF | oc create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sonar-data-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    namespace: sonarqube
    kind: PersistentVolumeClaim
    name: sonarqube-data
  nfs:
    path: /exports/sonar
    server: 192.168.1.30
EOF

cat <<EOF | oc create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sonardb-pv
spec:
  capacity:
    storage: 5Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    namespace: sonarqube
    kind: PersistentVolumeClaim
    name: sonardb
  nfs:
    path: /exports/sonardb
    server: 192.168.1.30
EOF

cat <<EOF | oc create -f -
apiVersion: v1
kind: PersistentVolume
metadata:
  name: sonar-ext-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  claimRef:
    namespace: sonarqube
    kind: PersistentVolumeClaim
    name: sonarqube-ext
  nfs:
    path: /exports/sonar-ext-data
    server: 192.168.1.30
EOF

# create template
oc create -f https://raw.githubusercontent.com/pedio/sonarqube/master/sonarqube-deployment-template.yml

# create app
oc new-app -f https://raw.githubusercontent.com/pedio/sonarqube/master/sonarqube-deployment-template.yml --param=SONARQUBE_MEMORY_LIMIT=2Gi


oc set resources dc/sonardb --limits=cpu=200m,memory=512Mi --requests=cpu=50m,memory=128Mi
oc set resources dc/sonarqube --limits=cpu=1,memory=2Gi --requests=cpu=200m,memory=256Mi


## already done in template.  This is an option to change settings
oc set volume dc/sonarqube --add --overwrite --name=sonar-data --mount-path=/opt/sonarqube/data/ --type persistentVolumeClaim --claim-name=sonarqube-data
oc set volume dc/sonarqube --add --overwrite --name=sonar-ext --mount-path=/opt/sonarqube/extensions --type persistentVolumeClaim --claim-name=sonarqube-ext-data

## already done in template.  This is an option to change settings 
oc set volume dc/sonarqube --add --overwrite --name=sonar-conf --mount-path=/opt/sonarqube/conf/sonar.properties --sub-path=sonar.properties --type configmap --configmap-name=sonar-conf


# post setup
curl https://admin:redhat@sonarqube.apps.devops.cheers.local/api/webhooks/create -X POST -d "name=jenkins&url=https://jenkins/sonarqube-webhook/"

oc label dc sonarqube "app.kubernetes.io/part-of"="sonarqube" --overwrite
oc label dc sonardb "app.kubernetes.io/part-of"="sonarqube" --overwrite

