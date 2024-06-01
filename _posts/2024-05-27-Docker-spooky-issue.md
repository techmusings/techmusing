---
layout: post
title: Docker Spooky issue
categories: [Blog]
tags: [docker,security,deployment,linux]
---
# Situation
---
One of my colleagues pinged for a site deployment support citing that all the binaries in one of the applicaton containers is getting deleted. The issue was reproducible and that was a good news.

# Task
---
The task in-hand was to:

1. Reproduce the issue
2. Identify the root-cause of why it is happening
3. Look at the ways to eliminate the root-cause
4. Take steps to avoid/automatically detect such issues in future

# Action
---
1. Since the issue was reproducible, first step was easy. Few observations:
   * The issue was getting reproduced only after container restart.
   * Interestingly random files were getting deleted.
   * All the files deleted were either binaries or shell scripts i.e. executables.
   
2. Just to isolate the issues multiple hit and tries were done. Few trials and learnings:
   * Dockerfile and docker compose was checked for any possibilities
   * The entry task and configuration was changed so that container remains running even though the binaries are not running
   * To observe any change in behaviour, volume mounting of the binaries and shell scripts was done on host m/c without any success.
   * Docker logs were not helpful
   * Journactl logs were also not helpful and didn't provide any clues
   * Restarting docker daemon also didn't changed anything

3. All the permissions of the folder from where binaries were getting deleted was changed to normal user (non-root user) along with launching the container using the same user. Few observations:
   * The permission to the folder was changed to 555 i.e. read and execute for all the users.
   * Observed that the folder permissions are getting changed from 'im' user to 'ugadmin' user along with write permission change also
4. With the above observation, it was quite evident that there is some external entity which is deleting the binaries/scripts. Few actions:

   * It is time to track that who is changing the folder permission. The only was is to observe the folder i.e. audit the folder for event.
   * The best way to do it is to use `auditctl` on the folder.
   * Since the folder was inside container and container base OS images are minimalisitic, `auditctl` service was missing inside the container. So had to host mount the folder.
   * Started observing the folder for any change using following command:

     ```
     auditctl -w /prd -p wa -k file_delete
     ```

     where

     * `-w /prd/im`: **W**atch the folder `/prd/im`
     * `-p wa`: **P**ermissions filter for the above watch.`w`for write. Similarly`r`for read,`x`for execute,`a` for append.``
     * `-k file_delete`: Filter **k**ey on the above use case i.e. watch with given permissions. This `file_delete` would act as filterkey i.e. text string (max 31 bytes long). It can uniquely identify the audit records produced by the watch. You need to use password-file string or phrase while searching audit logs.
5. After executing the above command, container was deployed and restarted to reproduce the issue an capture by `auditctl`.
6. Once the issue was redproduced, the audit records where searched using `ausearch` command below:

```
ausearch -i -k file_delete
```

where

* `-i`:**I**nterpret numerical entities into textual form. E.g. it would show account name instead of uid.
* `-k`:**K**eyword based search where keyword was created during `auditctl`monitoring

# Result
---
After the `ausearch`, auditlogs suggested that a process named `s1-agent` was deleting the files from the given folder. Further digging gave the information that the parent application named `Sentinelone` was the main culprit launching this process from the path `/opt/sentinelone/bin` and deleting the file from the given folder.

```
7343 type=PROCTITLE msg=audit(05/23/2024 18:43:35.942:155878) : proctitle=s1-agent
7344 type=PATH msg=audit(05/23/2024 18:43:35.942:155878) : item=1 name=/prd/ProcessConf/DeployConfig.sh inode=67360392 dev=fd:08 mode=file,640 ouid=ugadmin ogid=ugadmin rdev=00:     00 nametype=DELETE cap_fp=none cap_fi=none cap_fe=0 cap_fver=0 cap_frootid=0
```
