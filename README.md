# Active-Directory-Security-Anomaly-Detection

# Title: Enhancing Active Directory Security Through Machine Learning-Driven Anomaly Detection of Authentication Events

# Abstract: 
Organizations face sophisticated authentication attacks targeting Windows Active 
Directory infrastructure that bypass traditional signature-based security systems. 
This research developed a practical machine learning framework for identifying 
authentication anomalies through systematic event log analysis.

The methodology involved constructing a comprehensive Active Directory laboratory 
environment replicating enterprise-scale configurations, including multiple domain 
controllers, client workstations, and realistic organizational policies. Data 
collection systematically captured Windows Security Event Logs over extended 
periods, focusing on critical authentication events: successful logons (4624), 
failed attempts (4625), Kerberos ticket requests (4768, 4769), and NTLM 
authentications (4776).

Automated scripting was used to generate realistic baseline user behaviours 
simulating typical organizational workflows, varied login patterns, and standard 
resource access sequences. Feature engineering transformed raw event data into 
meaningful representations — temporal characteristics, behavioural patterns, 
credential usage metrics, and contextual authentication attributes.

The core architecture implemented a hybrid approach combining **autoencoder neural 
networks** for learning complex authentication behaviour patterns from normal 
operational data with **isolation forest algorithms** for efficiently identifying 
outlying authentication sequences indicative of potential threats.

This work delivers a deployable solution addressing real-world Active Directory 
security challenges through practical machine learning implementation, enhancing 
organizational capability to detect sophisticated authentication-based attacks.

## Keywords
Active Directory security, machine learning, anomaly detection, authentication 
attacks, Windows Event Logs, autoencoder, isolation forest, cybersecurity, 
enterprise security, threat detection

