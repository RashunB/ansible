---
title: Documentation References
tags: [#resources, #learning, #documentation]
created: 2026-02-06
---

# Documentation References

## Official Documentation

### Ansible
- **[Ansible Documentation](https://docs.ansible.com/)** - Comprehensive official docs
- **[Ansible Best Practices](https://docs.ansible.com/ansible/latest/tips_tricks/ansible_tips_tricks.html)** - Recommended patterns
- **[Ansible Module Index](https://docs.ansible.com/ansible/latest/collections/index.html)** - All available modules
- **[Jinja2 Templates](https://jinja.palletsprojects.com/)** - Template documentation

**Community Resources:**
- [Ansible Galaxy](https://galaxy.ansible.com/) - Shared roles and collections
- [Ansible Semaphore](https://www.ansible-semaphore.com/) - Web UI for Ansible
- [Reddit: r/ansible](https://www.reddit.com/r/ansible/)

---

### ZFS / OpenZFS
- **[OpenZFS Documentation](https://openzfs.github.io/openzfs-docs/)** - Primary reference
- **[Oracle ZFS Administration Guide](https://docs.oracle.com/cd/E19253-01/819-5461/)** - Detailed administration
- **[FreeBSD Handbook: ZFS](https://docs.freebsd.org/en/books/handbook/zfs/)** - Excellent tutorial
- **[Ubuntu ZFS Guide](https://ubuntu.com/tutorials/setup-zfs-storage-pool)** - Ubuntu-specific

**Advanced Topics:**
- [Aaron Toponce's ZFS Series](https://pthree.org/2012/04/17/install-zfs-on-debian-gnulinux/) - Deep dive blog series
- [JRS Systems Blog](https://jrs-s.net/category/open-source/zfs/) - Performance tuning
- [ZFS Best Practices (Klara Systems)](https://klarasystems.com/articles/openzfs-best-practices-guide/)

---

### systemd
- **[systemd Man Pages](https://www.freedesktop.org/software/systemd/man/)** - Authoritative reference
- **[systemd.exec](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)** - Service hardening options
- **[systemd for Administrators](https://www.freedesktop.org/wiki/Software/systemd/)** - Tutorial series by Lennart Poettering

**Security Guides:**
- [Securing systemd Services](https://www.ctrl.blog/entry/systemd-service-hardening.html)
- [systemd Security Features](https://www.freedesktop.org/software/systemd/man/systemd.exec.html#Security)

---

### PostgreSQL
- **[PostgreSQL Documentation](https://www.postgresql.org/docs/)** - Official docs (v16)
- **[PostgreSQL Tutorial](https://www.postgresqltutorial.com/)** - Beginner-friendly
- **[PostgreSQL Performance](https://wiki.postgresql.org/wiki/Performance_Optimization)** - Tuning guide
- **[Postgres Weekly](https://postgresweekly.com/)** - Newsletter

**Advanced Resources:**
- [Cybertec PostgreSQL Blog](https://www.cybertec-postgresql.com/en/blog/)
- [2ndQuadrant Blog](https://www.2ndquadrant.com/en/blog/)
- [Use The Index, Luke!](https://use-the-index-luke.com/) - SQL indexing guide

---

### Linux Security
- **[Linux Kernel Security](https://www.kernel.org/doc/html/latest/security/index.html)** - Kernel security subsystems
- **[Linux Capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html)** - Capability-based security
- **[seccomp](https://man7.org/linux/man-pages/man2/seccomp.2.html)** - Syscall filtering
- **[Namespaces](https://man7.org/linux/man-pages/man7/namespaces.7.html)** - Process isolation

**Security Frameworks:**
- [CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/) - Security baselines
- [NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)
- [OWASP Top 10](https://owasp.org/www-project-top-ten/)

---

## Books

### Infrastructure as Code
- **[Infrastructure as Code, 3rd Edition (Kief Morris)](https://www.oreilly.com/library/view/infrastructure-as-code/9781098114664/)**
  - Definitive guide to IaC patterns
  - Covers Terraform, Ansible, CloudFormation
  - Best practices and anti-patterns

- **[Ansible for DevOps (Jeff Geerling)](https://www.ansiblefordevops.com/)**
  - Practical, hands-on approach
  - Real-world examples
  - Available free online

- **[The Terraform Book (James Turnbull)](https://terraformbook.com/)**
  - Complementary to Ansible
  - Infrastructure provisioning patterns

---

### Site Reliability Engineering
- **[Site Reliability Engineering (Google)](https://sre.google/books/)**
  - Free online
  - Industry-defining practices
  - Monitoring, incident response, automation

- **[The Site Reliability Workbook (Google)](https://sre.google/workbook/table-of-contents/)**
  - Practical implementation guide
  - Case studies from Google
  - SLO/SLI frameworks

- **[Observability Engineering (O'Reilly)](https://www.oreilly.com/library/view/observability-engineering/9781492076438/)**
  - Modern monitoring approaches
  - Metrics, logs, traces

---

### Database Administration
- **[PostgreSQL Up & Running (O'Reilly)](https://www.oreilly.com/library/view/postgresql-up-and/9781491963401/)**
  - Quick start guide
  - Administration essentials
  - Performance tuning

- **[High Performance PostgreSQL (O'Reilly)](https://www.oreilly.com/library/view/high-performance-postgresql/9781491903063/)**
  - Advanced optimization
  - Query tuning
  - Hardware considerations

- **[Mastering PostgreSQL (Packt)](https://www.packtpub.com/product/mastering-postgresql-13-fourth-edition/9781800567498)**
  - Comprehensive coverage
  - Replication and HA
  - Advanced features

---

### Filesystems and Storage
- **[FreeBSD Mastery: ZFS (Michael W Lucas)](https://www.tiltedwindmillpress.com/product/fmzfs/)**
  - Practical ZFS administration
  - Troubleshooting guide
  - Real-world scenarios

- **[ZFS Administration Guide (Oracle)](https://docs.oracle.com/cd/E19253-01/819-5461/)**
  - Official reference (Solaris-based but applicable)
  - Comprehensive coverage

---

### Linux Administration
- **[The Linux Command Line (William Shotts)](https://linuxcommand.org/tlcl.php)**
  - Free online
  - Beginner to intermediate
  - Shell scripting

- **[UNIX and Linux System Administration Handbook (Nemeth et al.)](https://www.admin.com/)**
  - Industry standard reference
  - Covers everything from basics to advanced
  - Regularly updated

- **[How Linux Works (Brian Ward)](https://nostarch.com/howlinuxworks3)**
  - Understanding internals
  - Boot process, networking, filesystems

---

## Video Courses

### Ansible
- **[Learn Ansible (Jeff Geerling YouTube)](https://www.youtube.com/playlist?list=PL2_OBreMn7FqZkvMYt6ATmgC0KAGGJNAN)**
  - Free, high-quality series
  - Hands-on examples

- **[Ansible for the Absolute Beginner (KodeKloud)](https://kodekloud.com/courses/ansible-for-the-absolute-beginner/)**
  - Interactive labs
  - Beginner-friendly

- **[Red Hat Ansible Automation (Pluralsight)](https://www.pluralsight.com/courses/red-hat-ansible-automation-getting-started)**
  - Professional training
  - Certification preparation

---

### Linux & System Administration
- **[Linux Foundation Training](https://training.linuxfoundation.org/)**
  - Official Linux Foundation courses
  - LFCS/LFCE preparation

- **[Linux Academy / A Cloud Guru](https://www.pluralsight.com/cloud-guru)**
  - Hands-on labs
  - Learning paths for different roles

---

### Databases
- **[PostgreSQL Query Optimization (Pluralsight)](https://www.pluralsight.com/courses/postgresql-query-optimization)**
  - Performance tuning
  - EXPLAIN analysis

- **[PostgreSQL Administration (Udemy)](https://www.udemy.com/course/postgresql-database-administration/)**
  - Comprehensive administration
  - Backup and recovery

---

## Blogs and Newsletters

### DevOps and Infrastructure
- **[Hacker News](https://news.ycombinator.com/)** - Tech news and discussions
- **[DevOps Weekly](https://www.devopsweekly.com/)** - Curated newsletter
- **[SRE Weekly](https://sreweekly.com/)** - SRE-focused newsletter
- **[Last Week in AWS](https://www.lastweekinaws.com/)** - Cloud infrastructure news

### Specific Technologies
- **[Postgres Weekly](https://postgresweekly.com/)** - PostgreSQL news
- **[ZFS Mailing Lists](https://openzfs.github.io/openzfs-docs/Project%20and%20Community/Mailing%20Lists.html)** - Official discussions
- **[Ansible Collaborative](https://www.ansible.com/blog)** - Official Ansible blog

### Personal Blogs (High Quality)
- **[Julia Evans](https://jvns.ca/)** - Zines, Linux, debugging
- **[Brendan Gregg](https://www.brendangregg.com/blog/)** - Performance analysis
- **[Aaron Toponce](https://pthree.org/)** - ZFS, Linux administration
- **[Jan-Piet Mens](https://jpmens.net/)** - Ansible, automation

---

## Case Studies

### Infrastructure at Scale
- **[Netflix Tech Blog](https://netflixtechblog.com/)**
  - Chaos engineering
  - Microservices
  - CDN infrastructure

- **[Cloudflare Blog](https://blog.cloudflare.com/)**
  - DDoS mitigation
  - Edge computing
  - Performance optimization

- **[GitHub Engineering](https://github.blog/category/engineering/)**
  - Git at scale
  - Database sharding
  - Incident response

---

### Startup Infrastructure
- **[GitLab Handbook: Infrastructure](https://about.gitlab.com/handbook/engineering/infrastructure/)**
  - Transparent infrastructure docs
  - IaC practices
  - Monitoring and observability

- **[Basecamp: Getting Real](https://basecamp.com/gettingreal)**
  - Small team operations
  - Boring technology
  - Sustainable practices

---

### Homelab and Self-Hosting
- **[r/homelab](https://www.reddit.com/r/homelab/)** - Community discussions
- **[r/selfhosted](https://www.reddit.com/r/selfhosted/)** - Self-hosting software
- **[Awesome Selfhosted](https://github.com/awesome-selfhosted/awesome-selfhosted)** - Software list

- **[TrueNAS Community](https://www.truenas.com/community/)** - ZFS-focused
- **[Proxmox Forum](https://forum.proxmox.com/)** - Virtualization and ZFS

---

## Tools and Utilities

### Ansible Ecosystem
- **[Ansible Lint](https://ansible-lint.readthedocs.io/)** - Code quality checking
- **[Molecule](https://molecule.readthedocs.io/)** - Testing framework
- **[Ansible Semaphore](https://www.ansible-semaphore.com/)** - Web UI
- **[AWX](https://github.com/ansible/awx)** - Open-source Ansible Tower

---

### ZFS Tools
- **[Sanoid](https://github.com/jimsalterjrs/sanoid)** - Snapshot management
- **[Syncoid](https://github.com/jimsalterjrs/sanoid)** - ZFS replication
- **[ZFS Exporter](https://github.com/pdf/zfs_exporter)** - Prometheus metrics
- **[zfs-auto-snapshot](https://github.com/zfsonlinux/zfs-auto-snapshot)** - Automated snapshots

---

### Monitoring and Observability
- **[Prometheus](https://prometheus.io/docs/introduction/overview/)** - Metrics collection
- **[Grafana](https://grafana.com/docs/)** - Visualization
- **[Loki](https://grafana.com/docs/loki/latest/)** - Log aggregation
- **[Alertmanager](https://prometheus.io/docs/alerting/latest/alertmanager/)** - Alert routing

---

### Database Tools
- **[pgAdmin](https://www.pgadmin.org/)** - PostgreSQL GUI
- **[PgBouncer](https://www.pgbouncer.org/)** - Connection pooler
- **[pgBackRest](https://pgbackrest.org/)** - Backup and restore
- **[pgloader](https://pgloader.readthedocs.io/)** - Data migration

---

### Security Tools
- **[systemd-analyze security](https://www.freedesktop.org/software/systemd/man/systemd-analyze.html)** - Service security scoring
- **[Lynis](https://cisofy.com/lynis/)** - Security auditing
- **[OpenSCAP](https://www.open-scap.org/)** - Compliance scanning
- **[fail2ban](https://www.fail2ban.org/)** - Intrusion prevention

---

## Standards and Frameworks

### DevOps Practices
- **[The Twelve-Factor App](https://12factor.net/)** - Application design principles
- **[DORA Metrics](https://dora.dev/research/)** - DevOps performance measurement
- **[GitOps Principles](https://opengitops.dev/)** - Git-based operations

---

### Security Standards
- **[CIS Benchmarks](https://www.cisecurity.org/cis-benchmarks/)** - Security configuration baselines
- **[NIST Cybersecurity Framework](https://www.nist.gov/cyberframework)** - Risk management
- **[OWASP](https://owasp.org/)** - Application security

---

### Cloud Native
- **[CNCF Landscape](https://landscape.cncf.io/)** - Cloud native technologies
- **[Kubernetes Documentation](https://kubernetes.io/docs/home/)** - Container orchestration
- **[Docker Documentation](https://docs.docker.com/)** - Containerization

---

## Certifications

### Linux
- **[Linux Foundation Certified SysAdmin (LFCS)](https://training.linuxfoundation.org/certification/linux-foundation-certified-sysadmin-lfcs/)**
  - Cost: $395
  - Duration: 2 hours
  - Prerequisites: None
  - Good for: System administration fundamentals

- **[Red Hat Certified System Administrator (RHCSA)](https://www.redhat.com/en/services/certification/rhcsa)**
  - Cost: $400
  - Duration: 2.5 hours
  - Prerequisites: None
  - Good for: Enterprise Linux administration

---

### Automation
- **[Red Hat Certified Specialist in Ansible Automation](https://www.redhat.com/en/services/training/ex374-red-hat-certified-specialist-ansible-automation-exam)**
  - Cost: $400
  - Duration: 4 hours
  - Prerequisites: RHCSA recommended
  - Good for: Ansible expertise validation

- **[HashiCorp Certified: Terraform Associate](https://www.hashicorp.com/certification/terraform-associate)**
  - Cost: $70.50
  - Duration: 1 hour
  - Prerequisites: None
  - Good for: Infrastructure provisioning

---

### Cloud and Containers
- **[Certified Kubernetes Administrator (CKA)](https://training.linuxfoundation.org/certification/certified-kubernetes-administrator-cka/)**
  - Cost: $395
  - Duration: 2 hours
  - Prerequisites: None (but Kubernetes experience recommended)
  - Good for: Container orchestration

- **[AWS Certified Solutions Architect](https://aws.amazon.com/certification/certified-solutions-architect-associate/)**
  - Cost: $150
  - Duration: 130 minutes
  - Prerequisites: None
  - Good for: Cloud architecture

---

### Databases
- **[PostgreSQL Associate Certification (PGCA)](https://www.enterprisedb.com/course/postgresql-associate-certification)**
  - Cost: Free
  - Self-paced
  - Good for: PostgreSQL fundamentals

- **[PostgreSQL Certified Professional (PGCP)](https://www.enterprisedb.com/course/postgresql-certified-professional)**
  - Cost: Varies
  - Advanced PostgreSQL knowledge
  - Good for: DBA roles

---

## Community Resources

### Forums and Q&A
- **[Stack Overflow](https://stackoverflow.com/)** - Programming Q&A
- **[Server Fault](https://serverfault.com/)** - System administration Q&A
- **[Database Administrators Stack Exchange](https://dba.stackexchange.com/)** - Database Q&A
- **[Unix & Linux Stack Exchange](https://unix.stackexchange.com/)** - Linux-specific

---

### Reddit Communities
- **[r/ansible](https://www.reddit.com/r/ansible/)** - Ansible discussions
- **[r/zfs](https://www.reddit.com/r/zfs/)** - ZFS help and discussions
- **[r/postgresql](https://www.reddit.com/r/PostgreSQL/)** - PostgreSQL community
- **[r/homelab](https://www.reddit.com/r/homelab/)** - Homelab projects
- **[r/selfhosted](https://www.reddit.com/r/selfhosted/)** - Self-hosting
- **[r/devops](https://www.reddit.com/r/devops/)** - DevOps practices
- **[r/sysadmin](https://www.reddit.com/r/sysadmin/)** - System administration

---

### Discord Servers
- **[HomelabOS Discord](https://discord.gg/homelabos)** - Homelab automation
- **[Self-Hosted Discord](https://discord.gg/self-hosted)** - Self-hosting community
- **[DevOps Discord](https://discord.gg/devops)** - DevOps discussions

---

## Related Notes

- [[Required Skills Matrix]]
- [[Learning Pathways]]
- [[Industry Standard Comparisons]]

---

**Maintenance Note:** 
This list should be reviewed quarterly and updated with new resources as they become available.

**Last Updated:** 2026-02-06
