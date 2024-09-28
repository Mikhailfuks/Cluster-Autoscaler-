package main

import (  "context",flag","fmt","log","time"}

}

// Configuration for the autoscaler
type Config struct {
 ClusterName string
 Namespace   string
 MinNodes    int
 MaxNodes    int
 TargetCPU   float64
 Cooldown    time.Duration
}

// Function to get a Kubernetes clientset
func getClientset(kubeconfig string) (*kubernetes.Clientset, error) {
 var config *rest.Config
 var err error
 if kubeconfig != "" {
  config, err = clientcmd.BuildConfigFromFlags("", kubeconfig)
 } else {
  config, err = rest.InClusterConfig()
 }
 if err != nil {
  return nil, err
 }
 clientset, err := kubernetes.NewForConfig(config)
 if err != nil {
  return nil, err
 }
 return clientset, nil
}

// Function to check CPU utilization for the cluster
func getCPUUtilization(clientset *kubernetes.Clientset, namespace string) (float64, error) {
 // Get all nodes
 nodes, err := clientset.CoreV1().Nodes().List(context.Background(), metav1.ListOptions{})
 if err != nil {
  return 0, err
 }

 // Calculate total CPU capacity and total CPU usage
 totalCPUCapacity := 0.0
 totalCPUUsage := 0.0
 for _, node := range nodes.Items {
  // Get CPU capacity from the node's resources
  capacity, ok := node.Status.Capacity["cpu"]
  if !ok {
   continue
  }
  totalCPUCapacity += capacity.Value()

  // Get CPU usage from the node's metrics
  allocatable, ok := node.Status.Allocatable["cpu"]
  if !ok {
   continue
  }
  totalCPUUsage += allocatable.Value()
 }

 // Calculate CPU utilization
 if totalCPUCapacity == 0 {
  return 0, fmt.Errorf("no CPU capacity found")
 }
 return (totalCPUUsage / totalCPUCapacity) * 100, nil
}

// Function to scale the cluster up or down
func scaleCluster(clientset *kubernetes.Clientset, namespace string, minNodes int, maxNodes int, targetCPU float64, currentCPU float64) error {
 // Determine if scaling is needed
 if currentCPU > targetCPU && minNodes < maxNodes {
  // Scale down if CPU utilization is above the target
  fmt.Println("Scaling down cluster...")
  _, err := clientset.AutoscalingV2beta2().
   HorizontalPodAutoscalers(namespace).

   Update(context.Background(), &runtime.Unknown{Raw: []byte({"apiVersion": "autoscaling/v2beta2", "kind": "HorizontalPodAutoscaler", "metadata": {"name": "cluster-autoscaler"}, "spec": {"minReplicas": + fmt.Sprintf("%d", minNodes) + }})}, metav1.UpdateOptions{})
  if err != nil {
   return err
  }
  minNodes--
 } else if currentCPU < targetCPU && minNodes < maxNodes {
  // Scale up if CPU utilization is below the target
  fmt.Println("Scaling up cluster...")
  _, err := clientset.AutoscalingV2beta2().
   HorizontalPodAutoscalers(namespace).
   Update(context.Background(), &runtime.Unknown{Raw: []byte({"apiVersion": "autoscaling/v2beta2", "kind": "HorizontalPodAutoscaler", "metadata": {"name": "cluster-autoscaler"}, "spec": {"minReplicas": + fmt.Sprintf("%d", maxNodes) + }})}, metav1.UpdateOptions{})
  if err != nil {
   return err
  }
  maxNodes++
 }
 return nil
}

func main() {
 // Parse command-line flags
 kubeconfig := flag.String("kubeconfig", "", "Path to kubeconfig file")
 clusterName := flag.String("cluster-name", "", "Name of the Kubernetes cluster")
 namespace := flag.String("namespace", "default", "Namespace for the cluster autoscaler")
 minNodes := flag.Int("min-nodes", 2, "Minimum number of nodes in the cluster")
 maxNodes := flag.Int("max-nodes", 10, "Maximum number of nodes in the cluster")
 targetCPU := flag.Float64("target-cpu", 70, "Target CPU utilization percentage")
 cooldown := flag.Duration("cooldown", 10*time.Second, "Cooldown period between scaling operations")
 flag.Parse()

 // Get the Kubernetes clientset
 clientset, err := getClientset(*kubeconfig)
 if err != nil {
  log.Fatal("Error getting Kubernetes clientset:", err)
 }

 // Configuration for the autoscaler
 config := Config{
  ClusterName: *clusterName,
  Namespace:   *namespace,
  MinNodes:    *minNodes,
  MaxNodes:    *maxNodes,
  TargetCPU:   *targetCPU,
  Cooldown:    *cooldown,
 }

 // Continuously monitor CPU utilization and scale the cluster
 for {
  // Get current CPU utilization
  currentCPU, err := getCPUUtilization(clientset, config.Namespace)
  if err != nil {
   log.Println("Error getting CPU utilization:", err)
   time.Sleep(config.Cooldown)
   continue
  }

  // Scale the cluster up or down based on CPU utilization
  err = scaleCluster(clientset, config.Namespace, config.MinNodes, config.MaxNodes, config.TargetCPU, currentCPU)
  if err != nil {
   log.Println("Error scaling cluster:", err)
  }

  // Sleep for the cooldown period
  time.Sleep(config.Cooldown)
 }
}
