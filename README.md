# Readme for SAS Viya Admin Runbook Ideas

This readme document will provide information as well as suggested usage for considerations and ideas that may be helpful when administering SAS Viya:
- This is intended to plant a seed for ideas you may consider or commands you may want to automate
- There is no warranty, implied or otherwise
- These assets are solely meant to be a guide which, hopefully, provides helpful information and/or advice for administering SAS Viya

- [Readme for SAS Viya Admin Runbook Ideas](#readme-for-sas-viya-admin-runbook-ideas)
  - [Contributions](#contributions)
  - [Prerequisite Information:](#prerequisite-information)
  - [Background Information:](#background-information)
  - [Assumptions:](#assumptions)
      - [SAS-Viya-ARK](#sas-viya-ark)
  - [SAS Viya Service(s) Management:](#sas-viya-services-management)
  - [SAS Viya Host Monitoring:](#sas-viya-host-monitoring)
  - [SAS Viya Logging:](#sas-viya-logging)
  - [Tuning](#tuning)
  - [Libraries:](#libraries)
  - [Publishing Destinations:](#publishing-destinations)
  - [Security:](#security)
  - [License](#license)
  - [Support](#support)


## Contributions

We welcome all contributions! Please read [CONTRIBUTING.md](CONTRIBUTING.md) for details on how to submit contributions to this project.


## Prerequisite Information:

The following assets provide important background information as you administer SAS Viya.

The official [SAS Viya 3.5 Administration Guide](https://go.documentation.sas.com/?cdcId=calcdc&cdcVersion=3.5&docsetId=calwlcm&docsetTarget=home.htm&locale=en)

SAS Education Provided Viya Administration Training:
- [SAS Viya Administration Essentials](https://support.sas.com/edu/schedules.html?ctry=us&crs=SAVI)
- [SAS Viya Administration: Enhanced Tasks](https://support.sas.com/edu/schedules.html?ctry=us&crs=SAVA)
 
Here’s a [Viya Admin White Paper / Checklist](https://support.sas.com/resources/papers/checklist-sas-viya-administration-tasks.pdf) that may be helpful.
 
Here’s a [presentation from last year from a partner](https://www.sas.com/content/dam/SAS/support/en/sas-global-forum-proceedings/2019/3986-2019.pdf), I have not read it in detail, but skimmed it and it seemed relevant.
 
Here’s a [presentation from a few years ago from a conference](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=9&cad=rja&uact=8&ved=2ahUKEwjz3t3MzO3oAhVEl3IEHYqIAMwQFjAIegQIAhAB&url=https%3A%2F%2Fcommunities.sas.com%2Fkntur85557%2Fattachments%2Fkntur85557%2Fnordic-blog%2F815%2F1%2F96.%2520SAS%2520User%2520Forum%25202017_SAS%2520Viya%2520-%2520Architecture.pdf&usg=AOvVaw34Ber9XafUONnLAKAb2XCE), and much of the architecture holds true today
 

## Background Information:

As you may have your own unique IT governance practices, this document does not intent to overstep into your operations.

The author assumes that you will want to perform routine operations, the following is a limited list of example operations:
- restart the hosts on a regular cadence
  - note: in the authors experience, the operational focus was on Mean Time to Recover (MTTR), so emphasis was placed on regular monthly host restarts where multiple-month uptime duration was less important than the concept of **recoverability proven through repetition**
- update the operating system as well as any software & packages on a regular cadence
- configure monitoring hooks into your operational monitoring applications & routines
- develop additional automation that may potentially reduce operational resource requirements


## Assumptions:

The author assumes the following (note that your environment differences should be accounted for):
- that you are running Red Hat Enterprise Linux 7.x (or an equivalent distribution)
  - SAS supports [other operating systems, for example RHEL 8.x](https://support.sas.com/en/documentation/third-party-software-reference/viya/35/support-for-operating-systems.html), please refer to the administration guide for alternative commands for your OS
- that you have mirrored the SAS RPM's down to a `repo` host, this could also be called a bastion host
- that Ansible is {still} installed and operable on that `repo` host
  - SAS [supports these versions of Ansible](https://support.sas.com/en/documentation/third-party-software-reference/viya/35/support-for-operating-systems.html#ansible)
- that you have access to the installation service account, perhaps the account was named `viyadep`
- that you have the ability to issue commands as `viyadep` and/or `sudo` -- or that you could impersonate (su) `viyadep` and/or `root`
- that you have a common installation directory where the current installation files are located:
  - note: that the author recommends a symlink:
`Install_Playbook -> /opt/sas/install/sas_viya_playbook` 
  - note: older SAS orders can be archived for reference, for example:

    ```
    [viyadep@<myhost> ~]$ ll /opt/sas/install/
    drwxr-xr-x. 20 viyadep viyadep 4096 Apr  9 11:17 sas_viya_playbook
    drwxr-xr-x. 17 viyadep viyadep 4096 Jul 22  2019 sas_viya_playbook-{order123}
    drwxr-xr-x. 21 viyadep viyadep 4096 Mar 31 14:47 sas_viya_playbook-{order456}
    [viyadep@tes-repo ~]$
    ```

- that your common installation directory has the following files that were used for the Viya deployment so that Ansible works the same way it did at deployment:
    ```
    ansible.cfg
    inventory.ini
    ```

#### SAS-Viya-ARK

- that you have cloned the [SAS Viya-ARK](https://github.com/sassoftware/viya-ark/releases) into your common install directory:
    ```
    cd /opt/Install_Playbook
    git clone https://github.com/sassoftware/viya-ark/archive/Viya35-ark-1.3.tar.gz

    [viyadep@<myhost> Install_Playbook]$ ll /opt/Install_Playbook/ | grep "viya-ark"
    drwxrwxr-x.   4 viyadep viyadep     147 Apr  1 17:20 viya-ark-Viya35-ark-1.3
    ```

Note: your Viya depoyment may either be single machine or distrbuted across various hosts.


## SAS Viya Service(s) Management:

The SAS Viya microservices work together to provide the SAS Viya application to end users.  The number of microservices that are installed will depending on the deployment and the licensed software.  As previously mentioned, these microservices can be installed on a single host or they can be distributed across hosts in order to achieve common host types.  The following table indicates an example of a common topology for a distributed environment.

| Host Type      | Host Function                  |
|----------------|--------------------------------|
| CAS_Controller | CAS Controller                 |
| CAS_Worker_%   | CAS Worker - %                 |
| SAS            | SAS Programming Runtime Engine |
| Microservices  | Microservices                  |
| ESP_MAS        | Real time services             |

As documented in the [SAS Viya 3.5 Administration Guide | General Servers and Services: Operate (Linux)](https://go.documentation.sas.com/?cdcId=calcdc&cdcVersion=3.5&docsetId=calchkadm&docsetTarget=n00003ongoingtasks00000admin.htm&locale=en), "There is a sequence for starting and stopping SAS Viya servers and services. You must follow this sequence to avoid operational issues."  

- the Operate document provides numerous examples for single machine deployments, and 
- it indicates to use `/etc/init.d` when managing all services, and
- it indicates to use `systemctl` when managing individual services:
    ```
    # to check status of all servers and services:
    sudo /etc/init.d/sas-viya-all-services status

    # to stop all servers and services:
    sudo /etc/init.d/sas-viya-all-services stop

    # to start all servers and services:
    # its recommended to use a tmux shell to ensure there are no interruptions with your session
    # there are many resources found through Google to learn how to install and use tmux
    sudo /etc/init.d/sas-viya-all-services start

    # to operate an individual service, build a command based on this example:
    sudo systemctl {status | stop | start | restart} sas-viya-server-or-service-default

    # for example, to check the status of a single service, like SAS Logon:
    sudo systemctl status sas-viya-saslogon-default
    ```

- the Operate document also states that for a distributed environment, you should use the [SAS Viya-ARK](#SAS-Viya-ARK), which contains a set of Ansible playbooks intended to be run from the `repo` host as the `viyadep` user:
    ```
    # to stop all services
    cd /opt/Install_Playbook
    ansible-playbook viya-ark-Viya35-ark-1.3playbooks/viya-mmsu/viya-services-stop.yml

    # to start all services
    # its recommended to use a tmux shell to ensure there are no interruptions with your session
    # there are many resources found through Google to learn how to install and use tmux
    cd /opt/Install_Playbook
    ansible-playbook viya-ark-Viya35-ark-1.3/playbooks/viya-mmsu/viya-services-start.yml
    ```


## SAS Viya Host Monitoring:

In addition to the monitoring capabilities provided in SAS Environment Manager, SAS understands that you may have your own enterprise monitoring capabilities.  SAS is fully supportive of these capabilities.  In accordance with your IT governance practices, you may want to consider monitoring for the following states at minimum:

- Disk (volume) utilization above 70% / 80% / 90%
  - For example, disk utilization above an 80% threshold may result in a Major Alert
  - For example, disk utilization above an 90% threshold may result in a Critical Alert
- Average CPU Load consistently above 70% / 80% / 90%
  - For example, CPU  consistently above an 80% threshold consistently over a prolonged period of time may result in a Minor Alert
- *Significant wait for IO, this measurement may be somewhat subjective as a standard, but using a utility like dstat should not indicate "significant" waits for IO (meaning the CPU is data starved)
  - Wait for IO may result in a review incident to determine further action
- Average RAM utilization consistently above 70% / 80% / 90%
  - For example, RAM utilization above an 80% threshold may result in a Minor Alert
- A failure HTTP Response Code indicating the SASLogon service is not operational
  - For example, if you get an HTTP response code that's not a 200 series code, that situation may result in a Critical Alert
  - a suggested command to check the response follows:

  ```
  curl -s -o /dev/null -w "%{http_code}" https://{host}.com/SASLogon/login
  200
  # that command should provide a successful response, in this case the response code was 200
  ```

- Based on the environment, you may also want to monitor for excessive SWAP usage


## SAS Viya Logging:

As documented in the [SAS Viya 3.5 Administration Guide | Logging](https://go.documentation.sas.com/?cdcId=calcdc&cdcVersion=3.5&docsetId=callogging&docsetTarget=titlepage.htm&locale=en), "In SAS Viya, logs are produced not just by applications and servers, but also by each SAS Viya service. In order to manage the large number of logs and to enable you to locate messages of interest, the operations infrastructure provides components to collect and store log messages."

Standard SAS Viya logs for each host are located at `/opt/sas/*/config/var/log/*/default/*`

Standard logs for 3rd party applications, such as Apache HTTPD, tend to follow the expected logging directory, e.g. `/var/log/httpd/`

Some administrators may desire more aggressive log rotation routines than the standard configurations.  The author has written on this topic and has also developed some examples of what these more aggressive routines may encompass:

- these examples are based on Ansible tasks that could be further automated
- these examples write out result(s) to file(s), but they could be modified to issue a delete by replacing the `>> *.txt` with `-delete`:

```
name: Perform the log audit | logsOver{{ duration }}DaysOld
hosts: sas_all
shell: "find /opt/sas/viya/config/var/log/*/default/* -mtime +{{ duration }} -type f >> {{ file_location }}/logsOver{{ duration }}DaysOld.txt"

name: Perform the zipped log audit | zipOver{{ duration }}DaysOld
hosts: sas_all
shell: "find /opt/sas/viya/config/var/log -maxdepth 1 -mtime +{{ duration }} -name 'log-*.zip' >> {{ file_location }}/zipOver{{ duration }}DaysOld.txt"
   
name: Perform the log audit | logsOver{{ size }}SizeMb
hosts: sas_all
shell: "find /opt/sas/viya/config/var/log/*/default/* -size +{{ size }}M -type f >> {{ file_location }}/logsOver{{ duration }}SizeMb.txt"
    
name: Perform the log audit | casOver{{ duration }}DaysOld
hosts: sas_all
shell: "find /opt/sas/viya/config/var/log/cas/default_audit/* -mtime +{{ duration }} -type f >> {{ file_location }}/casOver{{ duration }}DaysOld.txt"
      
name: Perform the log audit | evdmOver{{ duration }}DaysOld
hosts: Operations
shell: "find /opt/sas/viya/config/var/lib/evmsvrops/evdm/log/log -maxdepth 1 -mtime +{{ duration }} -name '*.zip' >> {{ file_location }}/evdmOver{{ duration }}DaysOld.txt"
  
name: Perform the log audit | httpdOver{{ duration }}DaysOld
hosts: httpproxy
shell: "find /var/log/httpd/ -maxdepth 1 -mtime +{{ duration }} >> {{ file_location }}/httpdOver{{ duration }}DaysOld.txt"
    
name: Perform the SPRE audit | SASSecurityCertificateFramework{{ duration }}DaysOld
hosts: Operations
shell: "find /opt/sas/spre/config/var/log/SASSecurityCertificateFramework/default -mtime +{{ duration }} -type f >> {{ file_location }}/SASSecurityCertificateFramework{{ duration }}DaysOld.txt"

name: Perform the SPRE audit | SpoolerDir{{ duration }}DaysOld
hosts: Operations
shell: "find /opt/sas/viya/config/var/spool/compsrv/default/ -type d -mtime +{{ duration }} >> {{ file_location }}/SpoolerDir{{ duration }}DaysOld.txt"
```


## Tuning

There are several options for tuning your environment as documented in [SAS Viya 3.5 Administration Guide | Tuning](https://go.documentation.sas.com/?cdcId=calcdc&cdcVersion=3.5&docsetId=caltuning&docsetTarget=p1lh5hrky37woyn1wkk94tu5jq8b.htm&locale=en)

Additional tuning advice may be provided by SAS Customer Advisory, SAS Technical Support, and/or SAS Consulting based on facts gathered from observing your environment.


## Libraries:
- creating libraries / library types
- authorizing libraries
- a best practice could that that all data must be library backed or it could be lost in the event of a service restart

I may write more here.


## Publishing Destinations:
- PATH caslib
- batch and real time

I may write more here.


## Security:

There are multiple configurations available to further secure your environment as documented in [SAS Viya 3.5 Administration Guide | Security](https://go.documentation.sas.com/?cdcId=calcdc&cdcVersion=3.5&docsetId=calsecwlcm&docsetTarget=home.htm&locale=en)

Some topics include:
- httpd security
- CAS security
- Encryption for Data at Rest or Data in Motion
- Authentication
  - For authentication, SAS Viya requires an LDAP connection and can also be configured with various methods for single-sign-on capabilities
- Authorization
  - For authorization, there should be a strategy for how to configure SAS Custom Groups with your AD group(s)
  - CASHostAccountRequired is typically a required configuration to ensure that all data written to the file system utilizes user credentials instead of service account credentials


## License

This project is licensed under the [Apache 2.0 License](LICENSE).


## Support 

We use GitHub for tracking bugs and feature requests. Please submit a GitHub issue or pull request for support.