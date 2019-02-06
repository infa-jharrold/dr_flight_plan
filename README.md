# DR Region Setup Flight Plan
#### Create CTP tickets in JIRA for the following purposes.

1. VPC peering needs to be in place for the primary and DR region. This is necessary for two purposes:
	* Backend cluster HA (e.g. 3 Kafka nodes in Primary region and 2 in DR region).
	* EKS Cluster which is usually setup in a separate VPC needs access to above Backend clusters which are usually in the same VPC (but different subnets) as the IICS VPC.


2. EKS is setup in DR regions with proper peering to the DR region.
	* Prometheus Operator must be installed
	* Sumologic Fluentd collector must be 


3. Consul is setup in DR region. Find out what the Consul endpoint is from CloudTrust.

4. Vault is setup in DR region. Find out what the Vault endpoint is from CloudTrust.

5. Get management VPC and subnet information from CloudTrust.

### Steps to follow once you have all of the above information

##### Deploy Jenkins in the management subnet 
1. Create a certificate via Hydrant for Jenkins. First generate a certificate signing request.

	* Sample:
	
	```
	openssl req -new -newkey rsa:2048 -nodes -out mdmnextci_usw2_cloudtrust_rocks.csr -keyout mdmnextci_usw2_cloudtrust_rocks.key -subj "/C=US/ST=California/L=Redwood City/O=Informatica/OU=MDM/CN=mdmnextci-usw2.cloudtrust.rocks"
	```

2. Create a Jenkins node in the DR region's management VPC preferrably with Terraform.
	
	* Generate a jenkins p12 type certificate once you receive your certificate from Hydrant. You will require the root CA which will be provided by Hydrant as well. See e.g. below
	
	```
	openssl pkcs12 -export -out jenkins_keystore.p12 -passout 'pass:changeit' -inkey mdmnextci_usw2_cloudtrust_rocks.key -in mdmnextci_usw2_cloudtrust_rocks.crt -certfile QuoVadis_Root_CA_2.crt -name mdmnextci-usw2.cloudtrust.rocks
	```

	> `mdmnextci_usw2_cloudtrust_rocks.crt` will be generated via Hydrant and you can save it as any name you like. Make sure you substitute the correct filename in the above command.

	> `QuoVadis_Root_CA_2.crt` will also be available via Hydrant and you can save it as any name you like. Make sure you substitute the correct filename in the above command.

	> `mdmnextci_usw2_cloudtrust_rocks.key` must match the key generated in Step #1

	> `mdmnextci-usw2.cloudtrust.rocks` must match the CN as specified in Step #1.   

	* Copy the generated keystore to the Jenkins host:
	```scp -i /path/of/jenkins/ssh/key jenkins_keystore.p12 USER@JENKINSHOST:/home/centos/```
	* On the Jenkins host, create keystore directory: ```mkdir /var/lib/jenkins/keystore/```
	* Move the keystore to the directory: ```mv /home/centos/jenkins_keystore.p12 /var/lib/jenkins/keystore/```
	* Import into Jenkins:

	```
	keytool -importkeystore -srckeystore jenkins_keystore.p12 -srcstorepass 'changeit' -srcstoretype PKCS12 -srcalias mdmnextci-usw2.cloudtrust.rocks -deststoretype JKS -destkeystore jenkins_keystore.jks -deststorepass 'changeit' -destalias mdmnextci-usw2.cloudtrust.rocks
	```

3. Edit `/etc/sysconfig/jenkins` and add/edit/update the following lines.
	
	```
	JENKINS_JAVA_OPTIONS="-Djava.awt.headless=true -Dmail.smtp.starttls.enable=true"
	JENKINS_JAVA_OPTIONS="$JENKINS_JAVA_OPTIONS -Djenkins.install.runSetupWizard=false"
	JENKINS_HTTPS_PORT="8443"
	JENKINS_HTTPS_KEYSTORE="/var/lib/jenkins/keystore/jenkins_keystore.jks"
	JENKINS_HTTPS_KEYSTORE_PASSWORD="changeit"
	JENKINS_HTTPS_LISTEN_ADDRESS="0.0.0.0"
	JENKINS_PORT="-1" # Don't change this until you are sure HTTPS is working.
	```

4. Restart the jenkins service: `service jenkins restart`
