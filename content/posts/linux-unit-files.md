---
title: Linux Service Unit File Format
date: 2023-01-09 17:00:22
tags: [linux]
---

### References

https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files



### [Unit] Section

`Description=`

- just description

`Documentation=`

- ususlly a link to the official website

`Requires=`

- lists any units upon which this unit essentially depends
- the current unit starts when the required units are actived successfully
- required units are started in parallel by default

`Wants=`

- similar to `Requires=`, but less strict
- The systemd will attempt to start any units listed by `Wants=` when the current unit is actived. If wanted units are not found or failed to start, the current unit will continue to function.
- Wanted units are started in parallel unless modified by other directives.

`BindsTo=`

- similar to `Requires=`, but also causes the current unit to stop when the associated unit terminates.

`Before=`

- specifes units that will not be started unitl the current unit is marked as started
- does not imply a dependency relationship

`After=`

- specifies units that should be started before the current unit

`Conflicts=`

- lists units that cannot be run at the same time as the current unit

`Condition...=`

- can be used to provide a generic unit file that will only be run when on appropriate systems

`Assert...=`

- similar to `Condition...`
- a negative result causes a failure with this directive




### [Service] Section

`Type=`

- **simple**: The main process of the service is specified in the start line.
- **forking**: The service forks a child process and the parent process exits almost immediately. This type tells the systemd the service is sitll running even though the parent process exited.
- **onshot**: The service will be short-lived and that systemd should wait for the process to exit before continuing on with other units. Used for one-off tasks.
- **dbus**: The unit will take a name on the D-Bus bus.
- **notify**: The service will issue a notification when it has finished staring up. The systemd process will wait for this to happen before proceeding to other units.
- **idle**: The serivce will not be run until all jobs are dispatched.



### [Install] Section

`WantedBy=`

- specifies how a unit should be enabled
- When enabled, a directory will be created within `/etc/systemd/system` named after the specified unit with `.wants` appended to the end. A symbolic link to the current unit will be created, creating the dependency.
- For example, if the current unit has `WantedBy=multi-user.target`, a directory called `multi-user.target.wants` will be created within `/etc/systemd/system` and a symbolic link to the current unit will be placed within.
- Disabling the unit removes the link and removes the dependency relationship.

`RequiredBy=`

- similar to the `WantedBy` directive, but instead specifies a required dependency that will cause the activation to fail if not met
- When enabled, a unit with this directive will create a directory ending with `.requires`

`Alias=`

- allows units to be enabled under another name as well

`Also=`

- allows units to be enabled or disabled as a set

`DefaultInstance=`

- For template units (covered later) which can produce unit instances with unpredictable names, this can be used as a fallback value for the name if an appropriate name is not provided.