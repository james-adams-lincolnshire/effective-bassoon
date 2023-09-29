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

# Function to retrieve resource information for a pod
get_pod_resources() {
  local pod="$1"
  local namespace="$2"

  # Get resource information for the pod
  resource_info=$(kubectl get pod "$pod" -n "$namespace" -o json 2>/dev/null)

  if [[ -z "$resource_info" ]]; then
    echo "$pod,$namespace,Error,Error,Error,Error,Error,Error"
    return
  fi

  # Initialize variables to aggregate resources across containers
  limits_cpu_total=0
  limits_memory_total=0
  requests_cpu_total=0
  requests_memory_total=0

  # Loop through all containers in the pod and aggregate their resource data
  containers=$(echo "$resource_info" | jq -r '.spec.containers[].name')
  for container in $containers; do
    container_resources=$(echo "$resource_info" | jq -r '.spec.containers[] | select(.name == "'$container'").resources')
    
    # Get resource limits and requests for this container
    limits_cpu=$(echo "$container_resources" | jq -r '.limits.cpu')
    limits_memory=$(echo "$container_resources" | jq -r '.limits.memory')
    requests_cpu=$(echo "$container_resources" | jq -r '.requests.cpu')
    requests_memory=$(echo "$container_resources" | jq -r '.requests.memory')

    # Accumulate resources across containers
    limits_cpu_total=$((limits_cpu_total + $(convert_to_millicores "$limits_cpu")))
    limits_memory_total=$((limits_memory_total + $(convert_to_mebibytes "$limits_memory")))
    requests_cpu_total=$((requests_cpu_total + $(convert_to_millicores "$requests_cpu")))
    requests_memory_total=$((requests_memory_total + $(convert_to_mebibytes "$requests_memory")))
  done

  # Get resource usage for the pod
  usage_data=$(kubectl top pod "$pod" -n "$namespace" | awk 'NR>1 {print $1, $2, $3, $4}' 2>/dev/null)

  # Check if any value retrieval failed and set those values to "Error"
  if [[ -z "$usage_data" ]]; then
    echo "$pod,$namespace,Error,Error,Error,Error,Error,Error"
    return
  fi

  # Calculate resource usage by summing values across all containers in the pod
  usage_cpu_total=0
  usage_memory_total=0
  while read -r line; do
    container=$(echo "$line" | awk '{print $1}')
    cpu_usage=$(echo "$line" | awk '{print $2}')
    memory_usage=$(echo "$line" | awk '{print $4}')

    usage_cpu_total=$((usage_cpu_total + $(convert_to_millicores "$cpu_usage")))
    usage_memory_total=$((usage_memory_total + $(convert_to_mebibytes "$memory_usage")))
  done <<< "$usage_data"

  # Print the aggregated information in CSV format
  echo "$pod,$namespace,$limits_cpu_total,$requests_cpu_total,$usage_cpu_total,$limits_memory_total,$requests_memory_total,$usage_memory_total"
}

# Print CSV header
echo "Pod,Namespace,Limits CPU m,Requests CPU m,Usage CPU m,Limits Memory MiB,Requests Memory MiB,Usage Memory MiB"

# Get a list of all pods across all namespaces
pods=$(kubectl get pods --all-namespaces -o custom-columns="NAMESPACE:.metadata.namespace,POD:.metadata.name" --no-headers)

# Iterate through the list of pods and retrieve resource information
while read -r line; do
  namespace=$(echo "$line" | awk '{print $1}')
  pod=$(echo "$line" | awk '{print $2}')
  get_pod_resources "$pod" "$namespace"
done <<< "$pods"
