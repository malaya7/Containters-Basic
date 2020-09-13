## Program to simulate a basic docker container
Build: go build -o container container.go
running the program with command "run" <./container run>  will re-run the current executable by passing child as the first argument.
passing Cloneflags to SysProcAttr structer will namespaces to the program. Next we need to changing the filesystem from host to another place.
syscall.PivotRoot("rootfs", "rootfs/oldrootfs") Tell the OS to move the current directory at `/` to `rootfs/oldrootfs`
os.Chdir("/") swap the new rootfs directory to `/`

## Containers Architecture
Container is a lightweight VM that uses the host OS kernel to create an isolated environment and share resources on the host machine. 

There are 3 imporant parts for a Container
### Namespaces
Namespaces provide the isolation needed to run multiple containers on one machine while giving each what appears like it’s own environment
1. PID: The pid namespace gives a process and its children their own view of a subset of the processes in the system. Think of it as a mapping table. When a process in a pid namespace asks the kernel for a list of processes, the kernel looks in the mapping table. If the process exists in the table the mapped ID is used instead of the real ID. If it doesn’t exist in the mapping table, the kernel pretends it doesn’t exist at all. The pid namespace makes the first process created within it pid 1 (by mapping whatever its host ID is to 1), giving the appearance of an isolated process tree in the container.

2. MNT: In a way, this one is the most important. The mount namespace gives the process’s contained within it their own mount table. This means they can mount and unmount directories without affecting other namespaces (including the host namespace). More importantly, in combination with the pivot_root syscall - as we’ll see - it allows a process to have its own filesystem. This is how we can have a process think it’s running on ubuntu, or busybox, or alpine — by swapping out the filesystem the container sees.

3. NET: The network namespace gives the processes that use it their own network stack. In general only the main network namespace (the one that the processes that start when you start your computer use) will actually have any real physical network cards attached. But we can create virtual ethernet pairs — linked ethernet cards where one end can be placed in one network namespace and one in another creating a virtual link between the network namespaces. Kind of like having multiple ip stacks talking to each other on one host. With a bit of routing magic this allows each container to talk to the real world while isolating each to its own network stack.

4. UTS: The UTS namespace gives its processes their own view of the system’s hostname and domain name. After entering a UTS namespace, setting the hostname or the domain name will not affect other processes.

5. IPC: The IPC Namespace isolates various inter-process communication mechanisms such as message queues. See the Namespace docs for more details.

6. USER: The user namespace was the most recently added, and is the likely the most powerful from a security perspective. The user namespace maps the uids a process sees to a different set of uids (and gids) on the host. This is extremely useful. Using a user namespace we can map the container's root user ID (i.e. 0) to an arbitrary (and unprivileged) uid on the host. This means we can let a container think it has root access - we can even actually give it root-like permissions on container-specific resources - without actually giving it any privileges in the root namespace. The container is free to run processes as uid 0 - which normally would be synonymous with having root permissions - but the kernel is actually mapping that uid under the covers to an unprivileged real uid. Most container systems don't map any uid in the container to uid 0 in the calling namespace: in other words there simply isn't a uid in the container that has real root permissions.

### CGroups
groups collect a set of process or task ids together and apply limits to them. Where namespaces isolate a process, cgroups enforce fair resource sharing between processes.

### Layered Filesystems
Layered filesystems amount to optimising the call to create a copy of the root filesystem for each container.
Btrfs uses copy on write at the filesystem layer. Aufs uses “union mounts”.

###### refrence https://www.infoq.com/articles/build-a-container-golang/
