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
    echo "Error"
  fi
}

# Function to convert memory values to Mebibytes (MiB)
convert_to_mebibytes() {
  local value="$1"
  # Convert "Gi" to MiB
  if [[ "$value" =~ [0-9]+Gi ]]; then
    echo "$((value * 1024))"  # Convert Gi to MiB
  elif [[ "$value" =~ [0-9]+Mi ]]; then
    echo "${value%Mi}"  # Remove 'Mi' suffix
  else
    echo "Error"
  fi
}

# Function to retrieve resource information for a container
get_container_resources() {
  local pod="$1"
  local container="$2"
  local namespace="$3"

  local resource_data
  resource_data=$(kubectl get pod "$pod" -n "$namespace" -o jsonpath="{.spec.containers[?(@.name=='$container')].resources}") || resource_data="Error"

  if [[ "$resource_data" == "Error" ]]; then
    echo "$pod,$namespace,$container,Error,Error,Error,Error,Error,Error"
    return
  fi

  # Get resource limits and requests
  limits_cpu=$(echo "$resource_data" | jq -r '.limits.cpu')
  limits_memory=$(echo "$resource_data" | jq -r '.limits.memory')
  requests_cpu=$(echo "$resource_data" | jq -r '.requests.cpu')
  requests_memory=$(echo "$resource_data" | jq -r '.requests.memory')

  # Get resource usage
  usage_data=$(kubectl top pod "$pod" -n "$namespace" | awk 'NR>1 {print $2, $3}')
  usage_cpu=$(echo "$usage_data" | awk '{print $1}')
  usage_memory=$(echo "$usage_data" | awk '{print $2}')

  # Remove alpha characters from memory values
  limits_memory_clean="${limits_memory//[^0-9GiMi]/}"
  requests_memory_clean="${requests_memory//[^0-9GiMi]/}"
  usage_memory_clean="${usage_memory//[^0-9GiMi]/}"

  # Convert CPU values to milli-cores
  limits_cpu_millicores=$(convert_to_millicores "$limits_cpu")
  requests_cpu_millicores=$(convert_to_millicores "$requests_cpu")
  usage_cpu_millicores=$(convert_to_millicores "$usage_cpu")

  # Convert memory values to Mebibytes (MiB)
  limits_memory_mebibytes=$(convert_to_mebibytes "$limits_memory_clean")
  requests_memory_mebibytes=$(convert_to_mebibytes "$requests_memory_clean")
  usage_memory_mebibytes=$(convert_to_mebibytes "$usage_memory_clean")

  # Print the information in CSV format
  echo "$pod,$namespace,$container,$limits_cpu_millicores,$requests_cpu_millicores,$usage_cpu_millicores,$limits_memory_mebibytes,$requests_memory_mebibytes,$usage_memory_mebibytes"
}

# Print CSV header
echo "Pod,Namespace,Container,Limits CPU m,Requests CPU m,Usage CPU m,Limits Memory MiB,Requests Memory MiB,Usage Memory MiB"

# Get a list of all containers in all pods across all namespaces
containers=$(kubectl get pods --all-namespaces -o custom-columns="NAMESPACE:.metadata.namespace,POD:.metadata.name,CONTAINER:.spec.containers[*].name" --no-headers)

# Iterate through the list of containers and retrieve resource information
while read -r line; do
  namespace=$(echo "$line" | awk '{print $1}')
  pod=$(echo "$line" | awk '{print $2}')
  container=$(echo "$line" | awk '{print $3}')
  get_container_resources "$pod" "$container" "$namespace"
done <<< "$containers"
