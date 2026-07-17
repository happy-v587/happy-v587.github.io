# 据说这是k8s历史上的第一个operator


> 本文由原作者归档自 CSDN，原文发布于 2021-02-04。原文链接：[据说这是k8s历史上的第一个operator](https://blog.csdn.net/LIUHUAN0520/article/details/113647187)。

[https://www.cnblogs.com/zhaowei121/p/10255540.html](https://www.cnblogs.com/zhaowei121/p/10255540.html) 根据这个文章的描述，我们可以看出，k8s历史上第一个opertaor雏形应该是 etcd -operator。

为了了解operator的原理，我找到了etcd-operator的代码仓库，然后根据commit时间，找到了一个简单易懂的operator-demo。看完代码让我惊叹，原来operator居然几百行代码就可以完成。

[https://github.com/coreos/etcd-operator/tree/0f998e0f17a810c8f1a549c76ddc69fe8711eac0](https://github.com/coreos/etcd-operator/tree/0f998e0f17a810c8f1a549c76ddc69fe8711eac0)。

附上代码膜拜一下，只是由于依赖的版本、仓库地址变化，现在不能直接运行起来。

```Go
package main

import (
	"encoding/json"
	"errors"
	"flag"
	"fmt"
	"log"
	"net/http"
	"strings"

	"github.com/pborman/uuid"
	"k8s.io/kubernetes/pkg/api"
	"k8s.io/kubernetes/pkg/client/restclient"
	"k8s.io/kubernetes/pkg/client/unversioned"
	"k8s.io/kubernetes/pkg/util/intstr"
)

var masterHost string

func init() {
	flag.StringVar(&masterHost, "master", "http://127.0.0.1:8080", "usage")
	flag.Parse()
}

type etcdClusterController struct {
	kclient *unversioned.Client
}

func (c *etcdClusterController) Run() {
	eventCh, errCh := monitorNewCluster()
	for {
		select {
		case event := <-eventCh:
			c.createCluster(event)
		case err := <-errCh:
			panic(err)
		}
	}
}

func (c *etcdClusterController) createCluster(event newCluster) {
	size := event.Size
	uuid := generateUUID()

	initialCluster := []string{}
	for i := 0; i < size; i++ {
		initialCluster = append(initialCluster, fmt.Sprintf("etcd%d-%s=http://etcd%d-%s:2380", i, uuid, i, uuid))
	}

	for i := 0; i < size; i++ {
		etcdName := fmt.Sprintf("etcd%d-%s", i, uuid)

		svc := makeEtcdService(etcdName, uuid)
		_, err := c.kclient.Services("default").Create(svc)
		if err != nil {
			panic(err)
		}
		// TODO: add and expose client port
		pod := makeEtcdPod(etcdName, uuid, initialCluster)
		_, err = c.kclient.Pods("default").Create(pod)
		if err != nil {
			panic(err)
		}
	}
}

type newCluster struct {
	Kind       string            `json:"kind"`
	ApiVersion string            `json:"apiVersion"`
	Metadata   map[string]string `json:"metadata"`
	Size       int               `json:"size"`
}

type Event struct {
	Type   string
	Object newCluster
}

func monitorNewCluster() (<-chan newCluster, <-chan error) {
	events := make(chan newCluster)
	errc := make(chan error, 1)
	go func() {
		resp, err := http.Get(masterHost + "/apis/coreos.com/v1/namespaces/default/etcdclusters?watch=true")
		if err != nil {
			errc <- err
			return
		}
		if resp.StatusCode != 200 {
			errc <- errors.New("Invalid status code: " + resp.Status)
			return
		}
		log.Println("start watching...")
		for {
			decoder := json.NewDecoder(resp.Body)
			var ev Event
			err = decoder.Decode(&ev)
			if err != nil {
				errc <- err
			}
			event := ev.Object
			log.Println("new cluster size:", event.Size)
			events <- event
		}
	}()

	return events, errc
}

func main() {
	c := &etcdClusterController{
		kclient: mustCreateClient(masterHost),
	}
	log.Println("etcd cluster controller starts running...")
	c.Run()
}

func mustCreateClient(host string) *unversioned.Client {
	cfg := &restclient.Config{
		Host:  host,
		QPS:   100,
		Burst: 100,
	}
	c, err := unversioned.New(cfg)
	if err != nil {
		panic(err)
	}
	return c
}

func generateUUID() string {
	return uuid.New()
}

func makeEtcdService(etcdName, uuid string) *api.Service {
	labels := map[string]string{
		"etcd_node": etcdName,
		"etcd_uuid": uuid,
	}
	svc := &api.Service{
		ObjectMeta: api.ObjectMeta{
			Name:   etcdName,
			Labels: labels,
		},
		Spec: api.ServiceSpec{
			Ports: []api.ServicePort{{
				Name:       "server",
				Port:       2380,
				TargetPort: intstr.FromInt(2380),
				Protocol:   api.ProtocolTCP,
			}},
			Selector: labels,
		},
	}
	return svc
}

func makeEtcdPod(etcdName, uuid string, initialCluster []string) *api.Pod {
	pod := &api.Pod{
		ObjectMeta: api.ObjectMeta{
			Name: etcdName,
			Labels: map[string]string{
				"app":       "etcd",
				"etcd_node": etcdName,
				"etcd_uuid": uuid,
			},
		},
		Spec: api.PodSpec{
			Containers: []api.Container{
				{
					Command: []string{
						"/usr/local/bin/etcd",
						"--name",
						etcdName,
						"--initial-advertise-peer-urls",
						fmt.Sprintf("http://%s:2380", etcdName),
						"--listen-peer-urls",
						"http://0.0.0.0:2380",
						"--listen-client-urls",
						"http://0.0.0.0:2379",
						"--advertise-client-urls",
						fmt.Sprintf("http://%s:2379", etcdName),
						"--initial-cluster",
						strings.Join(initialCluster, ","),
						"--initial-cluster-state",
						"new",
					},
					Name:  etcdName,
					Image: "gcr.io/coreos-k8s-scale-testing/etcd-amd64:3.0.4",
					Ports: []api.ContainerPort{
						{
							Name:          "server",
							ContainerPort: int32(2380),
							Protocol:      api.ProtocolTCP,
						},
					},
				},
			},
			RestartPolicy: api.RestartPolicyNever,
		},
	}
	return pod
}
```

