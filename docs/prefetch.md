# Proposal: A image prefetch mechanism based on container's startup I/O pattern

## Overview

Cache has been playing an important role in the whole architecture of ACI's [I/O flow](docs/images/image-flow.jpg "image data flow"). When there is no cache (cold start), however, the backend storage engine will still need to visit Registry frequently, and temporarily.

Prefetch is a common mechanism to avoid this situation. As it literally suggests, the key is to retrieve data in advance.

There are many ways to do prefetch, for instance, we can simply read extra data beyond the designated range of a Registry blob. That might be called as Expand Prefetch, and the expand ratio could be 2x, 4x, or even higher, if our network bandwidth is sufficient.

Another way is to [prioritize files and use landmarks](https://github.com/containerd/stargz-snapshotter/blob/master/docs/stargz-estargz.md#prioritized-files-and-landmark-files), which is already adopted in stargz. The storage engine runtime should prefetch the range where prioritized files are contained. And finally this information will be leveraged for increasing cache hit ratio and mitigating read overhead.

In this article we are about to discuss a new prefetch mechanism based on container's startup I/O pattern. This mechanism should work with ACI and overlaybd image format.

## Trace Prefetch

Since every single I/O request happens on user's own filesystem will eventually be mapped into one overlaybd's layer blob, we can then record all these I/O patterns from the layer blob's perspective. And that's why we call it Trace Prefetch.

Trace prefetch is time based, and it has greater granularity and predication accuracy than stargz. We don't mark a file, because user app might only need to read a small part of it in the beginning, simply prefetching the whole file would be less efficient. Instead, we replay the recorded trace, by the exact I/O pattern that happened before. Each pattern contains only necessary information, such as the offset and length of the blob being read.

Trace is stored as an independent image layer, and must always be the uppermost one. Neither image manifest nor container snapshotter needs to know if it is a trace layer, they just download and extract it as usual. The overlaybd backstore is responsible to recognize trace layer, and replay it accordingly.

## Terminology

### Record

Run a container based on the target image, persist I/O patterns during startup, and then dump them into a trace blob. The trace blob will be chained, and become the top layer.   

When building a new image from a base image, the old trace layer (if exists in the base image) must be removed. New trace layer might be added later, if recording is desired.

### Push 

Push command will save both data layer and trace layer to Registry. Trace layer looks transparent to it.

### Replay

After Recording and Pushing, users could pull and run the specific image somewhere else. Snapshotter's storage backend should load the trace blob, and replay I/O patterns for each layer blob.

## Example Usage

Suppose we have a docker runtime of containerd + overlaybd snapshotter
```
docker pull <overlaybd_image>

docker run --record-trace --record-time 10s <overlaybd_image> <container>

docker commit <container> <new_image>

docker push <new_image>
```