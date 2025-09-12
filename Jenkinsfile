pipeline {
    agent any

    environment {
        ANSIBLE_PLAYBOOK = "kafka-zookeeper-cluster/cluster-playbook.yml"
        INVENTORY       = "kafka-zookeeper-cluster/inventory.ini"
        KAFKA_BIN       = "/opt/kafka/kafka_2.13-3.7.0/bin"
        BROKERS         = "172.31.22.11:9092,172.31.16.250:9092,172.31.18.231:9092"
        TEST_TOPIC      = "ci-cd-test-topic"
        NODES           = "172.31.22.11 172.31.16.250 172.31.18.231"
        SSH_USER        = "ubuntu"   // change if your EC2 uses a different user
    }

    stages {
        stage('Checkout Code') {
            steps {
                echo "Checking out repository..."
                git branch: 'main', url: 'https://github.com/AnithaAnnem/my-kafka-task.git'
            }
        }

        stage('Ping Connectivity Check') {
            steps {
                echo "Pinging all Kafka/ZooKeeper nodes..."
                sh """
                    set -e
                    for node in ${NODES}; do
                        echo "Pinging \$node..."
                        ping -c 2 \$node || { echo "Node \$node is unreachable"; exit 1; }
                    done
                """
            }
        }

        stage('SSH Connectivity Check') {
            steps {
                echo "Checking SSH connectivity to all nodes..."
                sshagent(['my-kafka-ssh-key']) {
                    sh """
                        set -e
                        for node in ${NODES}; do
                            echo "Testing SSH to \$node..."
                            ssh -o StrictHostKeyChecking=no -o BatchMode=yes -o ConnectTimeout=5 ${SSH_USER}@\$node "echo SSH OK on \$node"
                        done
                    """
                }
            }
        }

        stage('Deploy Cluster with Ansible') {
            steps {
                echo "Running Ansible playbook to deploy Zookeeper + Kafka..."
                sshagent(['my-kafka-ssh-key']) {
                    sh "ansible-playbook -i ${INVENTORY} ${ANSIBLE_PLAYBOOK}"
                }
            }
        }

        stage('Create Test Topic') {
            steps {
                echo "Creating test topic..."
                sh """
                    set -e
                    ${KAFKA_BIN}/kafka-topics.sh --bootstrap-server ${BROKERS} \
                    --create --topic ${TEST_TOPIC} --partitions 3 --replication-factor 3 || echo 'Topic already exists'
                """
            }
        }

        stage('Produce Test Message') {
            steps {
                echo "Producing test message..."
                sh """
                    set -e
                    echo 'Hello from CI/CD pipeline!' | \
                    ${KAFKA_BIN}/kafka-console-producer.sh --broker-list ${BROKERS} --topic ${TEST_TOPIC}
                """
            }
        }

        stage('Consume Test Message') {
            steps {
                echo "Consuming test message..."
                sh """
                    set -e
                    ${KAFKA_BIN}/kafka-console-consumer.sh --bootstrap-server ${BROKERS} \
                    --topic ${TEST_TOPIC} --from-beginning --timeout-ms 5000
                """
            }
        }

        stage('Cluster Health Check') {
            steps {
                echo "Checking broker connectivity..."
                sh """
                    set -e
                    for broker in ${NODES}; do
                        ${KAFKA_BIN}/kafka-broker-api-versions.sh --bootstrap-server \$broker:9092
                    done
                """
            }
        }
    }

    post {
        success {
            echo "Kafka + Zookeeper cluster deployment and validation successful ✅"
        }
        failure {
            echo "❌ Deployment failed. Check logs."
        }
    }
}
