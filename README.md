# effective-bassoon

#!/bin/bash

# Function to convert CPU values to milli-cores
convert_to_millicores() {
  local value="$1"
  # Remove 'm' if it's present and multiply by 1000
  if [[ "$value" =~ [0-9]+m ]]; then
    echo "${value%m}"  # Remove 'm' suffix
  elif [[ "$value" =~ [0-9]+ ]]; then
    echo "$((value * 1000))"  # Convert to milli-cores
  else
    echo "Invalid"
  fi
}

# Function to retrieve resource information for a container
get_container_resources() {
  local pod="$1"
  local namespace="$2"
  local container="$3"

  # Get resource limits
  limits_cpu=$(kubectl get pod "$pod" -n "$namespace" -o=jsonpath="{.spec.containers[?(@.name=='$container')].resources.limits.cpu}")
  limits_memory=$(kubectl get pod "$pod" -n "$namespace" -o=jsonpath="{.spec.containers[?(@.name=='$container')].resources.limits.memory}")

  # Get resource requests
  requests_cpu=$(kubectl get pod "$pod" -n "$namespace" -o=jsonpath="{.spec.containers[?(@.name=='$container')].resources.requests.cpu}")
  requests_memory=$(kubectl get pod "$pod" -n "$namespace" -o=jsonpath="{.spec.containers[?(@.name=='$container')].resources.requests.memory}")

  # Get resource usage
  usage_cpu=$(kubectl top pod "$pod" -n "$namespace" | awk 'NR>1 {print $2}')
  usage_memory=$(kubectl top pod "$pod" -n "$namespace" | awk 'NR>1 {print $3}')

  if [[ -z "$limits_cpu" || -z "$limits_memory" || -z "$requests_cpu" || -z "$requests_memory" || -z "$usage_cpu" || -z "$usage_memory" ]]; then
    echo "Error: Failed to retrieve resource information for pod $pod in namespace $namespace."
    return
  fi

  # Remove 'm' from CPU values and convert to milli-cores
  limits_cpu_millicores=$(convert_to_millicores "$limits_cpu")
  requests_cpu_millicores=$(convert_to_millicores "$requests_cpu")
  usage_cpu_millicores=$(convert_to_millicores "$usage_cpu")

  # Remove alpha characters from memory values
  limits_memory_clean="${limits_memory//[^0-9]/}"
  requests_memory_clean="${requests_memory//[^0-9]/}"
  usage_memory_clean="${usage_memory//[^0-9]/}"

  # Print the information in a tab-separated format with headers
  echo -e "Pod\tNamespace\tContainer\tLimits (CPU m)\tRequests (CPU m)\tUsage (CPU m)\tLimits (Memory MiB)\tRequests (Memory MiB)\tUsage (Memory MiB)"
  echo -e "$pod\t$namespace\t$container\t$limits_cpu_millicores\t$requests_cpu_millicores\t$usage_cpu_millicores\t$limits_memory_clean\t$requests_memory_clean\t$usage_memory_clean"
  echo "---------------------------"
}

# Get a list of all containers in all pods across all namespaces
containers=$(kubectl get pods --all-namespaces -o custom-columns="NAMESPACE:.metadata.namespace,POD:.metadata.name,CONTAINER:.spec.containers[*].name" --no-headers | awk '{split($3,containers,","); for(i in containers) print $1,$2,containers[i]}')

# Iterate through the list of containers and retrieve resource information
while read -r line; do
  namespace=$(echo "$line" | awk '{print $1}')
  pod=$(echo "$line" | awk '{print $2}')
  container=$(echo "$line" | awk '{print $3}')
  echo "Retrieving resource information for container $container in pod $pod, namespace $namespace..."
  get_container_resources "$pod" "$namespace" "$container"
done <<< "$containers"
