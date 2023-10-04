# Singularity-build-yum-agent
singularity-ce version 4.0.0, yum, CentOS 7, 3.10.0-1160.76.1.el7.x86_64, RPM version 4.11.3

Tried to build singularity image using the "yum" bootstrap, using the example from: 
https://github.com/sylabs/singularity/blob/main/examples/centos/Singularity

```
BootStrap: yum
OSVersion: 7
MirrorURL: http://mirror.centos.org/centos-%{OSVERSION}/%{OSVERSION}/os/x86_64/
Include: yum

# If you want the updates (available at the bootstrap date) to be installed
# inside the container during the bootstrap instead of the General Availability
# point release (7.x) then uncomment the following line
#UpdateURL: http://mirror.centos.org/centos-%{OSVERSION}/%{OSVERSION}/updates/$basearch/


%runscript
    echo "This is what happens when you run the container..."


%post
    echo "Hello from inside the container"
    yum -y install vim-minimal
```

It gave me errot as the following: 

```
[feng@centos]$  singularity  build --fakeroot centos-base.sif base.def
INFO:    Starting build...
FATAL:   While performing build: conveyor failed to get: while checking rpm path: macro is not defined
```



Very confusing. Digged into it's source code: In file "internal/pkg/build/sources/conveyorPacker_yum.go",

```
func (c *YumConveyor) getRPMPath() (err error) {
        c.rpmPath, err = bin.FindBin("rpm")
        if err != nil {
                return fmt.Errorf("rpm is not in path: %v", err)
        }

        rpmDBBackend, err := rpm.GetMacro("_db_backend")
        if err != nil {
                return err
        }
        if rpmDBBackend != "bdb" {
                sylog.Warningf("Your host system is using the %s RPM database backend.", rpmDBBackend)
                sylog.Warningf("Bootstrapping of older distributions that use the bdb backend will fail.")
        }
        rpmDBPath, err := rpm.GetMacro("_dbpath")
        if err != nil {
                return err
        }
        // %{_var}/lib/rpm is the 'traditional' dbpath
        if rpmDBPath != `/var/lib/rpm` {
```
Then check the system rpm's macros

```
[feng]# rpm --eval '%{_rpmdir}'
/root/rpmbuild/RPMS
[feng]# rpm --eval '%{_db_backend}'
%{_db_backend}
[feng]# rpm --eval '%{_dbpath}'
/var/lib/rpm

```

Obviously,  the '%{_db_backend}' is not defined. Not sure what it really is, add it to " ~/.rpmmacros".

```
%_db_backend bdb
```
Then it works:

```

[feng@centos]$  singularity  build --fakeroot centos-base-orig.sif base-orig.def
INFO:    Starting build...
INFO:    Skipping GPG Key Import
INFO:    Adding owner write permission to build path: /tmp/build-temp-718041111/rootfs
INFO:    Running post scriptlet

```
