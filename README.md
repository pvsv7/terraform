#!/bin/bash

# Set the necessary variables
INSTANCE_ID="your-ec2-instance-id"  # Replace with your EC2 instance ID
NAMESPACE="db2metrics"

# Function to send custom CloudWatch metrics
send_metric() {
    local METRIC_NAME=$1
    local VALUE=$2
    local UNIT=$3

    aws cloudwatch put-metric-data --metric-name "$METRIC_NAME" --namespace "$NAMESPACE" --dimensions "InstanceId=$INSTANCE_ID" --value "$VALUE" --unit "$UNIT"
}

# 1. Get DB2 status (0 if active, 1 if inactive)
DB_STATUS=$(db2 get db status | grep "ACTIVE" > /dev/null && echo 0 || echo 1)

# 2. Get table space utilization percentage
TABLESPACE_UTIL=$(db2 "SELECT TBSP_UTILIZATION_PERCENT FROM SYSIBMADM.TBSP_UTILIZATION" | grep -Eo '[0-9]+(\.[0-9]+)?')

# 3. Get bufferpool hit ratio (example value; adjust query according to your setup)
BUFFERPOOL_HIT_RATIO=$(db2 "SELECT POOL_HIT_RATIO_PERCENT FROM SYSIBMADM.BP_HITRATIO" | grep -Eo '[0-9]+(\.[0-9]+)?')

# 4. Get percentage of log utilized
LOG_UTILIZED=$(db2 "SELECT TOTAL_LOG_USED_PERCENT FROM SYSIBMADM.LOG_UTILIZATION" | grep -Eo '[0-9]+(\.[0-9]+)?')

# Check if values were retrieved successfully
if [[ -z "$DB_STATUS" || -z "$TABLESPACE_UTIL" || -z "$BUFFERPOOL_HIT_RATIO" || -z "$LOG_UTILIZED" ]]; then
    echo "Failed to retrieve DB2 metrics."
    exit 1
fi

# Send DB2 status to CloudWatch
send_metric "DBStatus" "$DB_STATUS" "Count"

# Send table space utilization percentage to CloudWatch
send_metric "TableSpaceUtilizationPercentage" "$TABLESPACE_UTIL" "Percent"

# Send bufferpool hit ratio to CloudWatch
send_metric "BufferPoolHitRatio" "$BUFFERPOOL_HIT_RATIO" "Percent"

# Send percentage of log utilized to CloudWatch
send_metric "LogUtilizationPercentage" "$LOG_UTILIZED" "Percent"

# Notify success
echo "DB2 metrics successfully sent to CloudWatch."

---
#!/bin/bash

# Set the necessary variables
INSTANCE_ID="yi-058bd3ee3dd5261db"  # Replace with your EC2 instance ID
NAMESPACE="db2uptime"

# Get MySQL uptime in seconds
UPTIME=$(sudo mysql -u root -e "SHOW GLOBAL STATUS LIKE 'Uptime';" | grep 'Uptime' | awk '{print $2}')

# Check if we successfully retrieved uptime
if [[ -z "$UPTIME" ]]; then
    echo "Failed to retrieve MySQL uptime."
    exit 1
fi

# Publish custom metric to CloudWatch
aws cloudwatch put-metric-data --metric-name "dbuptime" --namespace "$NAMESPACE" --dimensions "InstanceId=$INSTANCE_ID" --value "$UPTIME" --unit "Seconds"

# Notify success
echo "MySQL Uptime: $UPTIME seconds successfully sent to CloudWatch."





