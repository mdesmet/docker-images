# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

ARG ARCH
FROM testing/almalinux9-oj17:unlabelled$ARCH

# INSTALL OPENLDAP
RUN yum -y install openssl openldap openldap-clients openldap-servers \
    # Cleanup
    && yum -y clean all && rm -rf /tmp/* /var/tmp/*

# COPY CONFIGURATION
COPY ./files /

# Run supervisord in background
RUN supervisord -c /etc/supervisord.conf && \
    /usr/bin/wait-for-slapd.sh && \
    ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/openldap/setup/modify_server.ldif && \
    ldapmodify -Y EXTERNAL -H ldapi:/// -f /etc/openldap/setup/ldap_disable_bind_anon.ldif && \
    # ldif are required in order to have inetOrgPerson object class
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/cosine.ldif && \
    ldapadd -Y EXTERNAL -H ldapi:/// -f /etc/openldap/schema/inetorgperson.ldif

# Generate a keystore.
# TODO: https://github.com/trinodb/trino/issues/8239
RUN keytool -genkey -alias coordinator -storepass testldap -keystore /etc/openldap/certs/coordinator.jks \
    -keypass testldap -keyalg RSA -sigalg SHA1withRSA -dname "CN=presto-master, OU=, O=, L=, S=, C=" -validity 100000

# Generate a certificate and CSR for the keystore. CN should match the hostname of trino-coordinator
RUN keytool -export -alias coordinator -storepass testldap -keystore /etc/openldap/certs/coordinator.jks \
    -file /etc/openldap/certs/coordinator.cer && \
    keytool -certreq -alias coordinator -storepass testldap -keystore /etc/openldap/certs/coordinator.jks \
    -file /etc/openldap/certs/coordinator.csr

# create a test CA and generate caroot.cer( root certificate of the CA ).
RUN openssl genrsa -out /etc/openldap/certs/cakey.pem 2048 && \
    openssl req -x509 -new -nodes -key /etc/openldap/certs/cakey.pem -sha256 -days 3650 -out /etc/openldap/certs/caroot.cer \
    -subj "/C=US/ST=Massachusetts/L=Boston/O=Teradata/OU=Finance/CN=teradata" \
    -addext "basicConstraints=CA:TRUE" \
    -addext "keyUsage=keyCertSign,cRLSign"

# sign the server certificate using the testCA
RUN openssl x509 -req -in /etc/openldap/certs/coordinator.csr -out /etc/openldap/certs/TestCA.cer -days 100000 \
    -CA /etc/openldap/certs/caroot.cer -CAkey /etc/openldap/certs/cakey.pem -CAserial /etc/openldap/certs/serial.txt

# Import the caroot.cer to the keystore and replace the previously generated self-signed cert with the new CA signed one in the keystore
# Note: The -alias name (coordinator) should be the same as the alias used when coordinator.cer was generated
RUN keytool -import -alias TestCA -storepass testldap -keystore /etc/openldap/certs/coordinator.jks \
    -noprompt -file /etc/openldap/certs/caroot.cer && \
    keytool -import -alias coordinator -storepass testldap -keystore /etc/openldap/certs/coordinator.jks \
    -file /etc/openldap/certs/TestCA.cer

# Import the root certificate of the issuer of the trino-coordinator certificate.
# This would be used by clients ( cli, jdbc ) to run tests
RUN keytool -import -alias caroot -storepass testldap -keystore /etc/openldap/certs/cacerts.jks -noprompt \
    -file /etc/openldap/certs/caroot.cer
