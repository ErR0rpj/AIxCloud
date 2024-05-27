Any container runtime works via Container runtime interface (CRI) in kubernetes other than docker.
Docker works via dockershim.
Containerd was deamon created for kubernetes but then it supports the CRI and kubernetes started maintaining it.
Docker support and dockershim has now being removed from kubernetes and containerd is now being maintained.