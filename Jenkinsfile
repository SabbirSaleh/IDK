pipeline {
    agent any

    environment {
        FABRIC_CFG_PATH = "${WORKSPACE}/config"
        CHAINCODE_NAME = "basic"
        CHAINCODE_VERSION = "1.0"
        CHANNEL_NAME = "mychannel"
        PEER0_ORG1_TLSCERT = "${WORKSPACE}/test-network/organizations/peerOrganizations/org1.example.com/tlsca/tlsca.org1.example.com-cert.pem"
        ORDERER_TLSCERT = "${WORKSPACE}/test-network/organizations/ordererOrganizations/example.com/tlsca/tlsca.example.com-cert.pem"
        PATH = "${WORKSPACE}/bin:${PATH}"
        CORE_PEER_ADDRESS = "localhost:7051" // Update if Jenkins runs in Docker
        ORDERER_ADDRESS = "localhost:7050"  // Update if Jenkins runs in Docker
    }

    stages {
        stage('Download Binaries') {
            steps {
                sh '''
                echo "Downloading Fabric binaries..."
                curl -sSL https://bit.ly/2ysbOFE | bash -s -- -d
                mv bin/* ${WORKSPACE}/bin/
                export PATH=${WORKSPACE}/bin:$PATH
                '''
            }
        }

        stage('Prepare MSP Directory') {
            steps {
                sh '''
                echo "Preparing MSP directory..."
                mkdir -p ${WORKSPACE}/test-network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/
                cp -r ~/fabric-samples/test-network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp \
                    ${WORKSPACE}/test-network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/
                '''
            }
        }

        stage('Prepare Config Directory') {
            steps {
                sh '''
                echo "Preparing config directory..."
                mkdir -p ${WORKSPACE}/config
                cp ~/fabric-samples/config/* ${WORKSPACE}/config/
                '''
            }
        }

        stage('Setup Environment') {
            steps {
                sh '''
                echo "Setting up environment..."
                docker --version
                peer version || echo "Peer binary not found"
                '''
            }
        }

        stage('Start Fabric Network') {
            steps {
                sh '''
                echo "Starting Fabric test network..."
                cd test-network
                ./network.sh down
                ./network.sh up createChannel -c ${CHANNEL_NAME}
                ./network.sh deployCC -ccn ${CHAINCODE_NAME} -ccp ../asset-transfer-basic/chaincode-go -ccl go
                '''
            }
        }

        stage('Package Chaincode') {
            steps {
                sh '''
                echo "Packaging chaincode..."
                peer lifecycle chaincode package ${CHAINCODE_NAME}.tar.gz \
                    --path ../asset-transfer-basic/chaincode-go \
                    --lang golang \
                    --label ${CHAINCODE_NAME}_${CHAINCODE_VERSION}
                '''
            }
        }

        stage('Install Chaincode') {
            steps {
                sh '''
                echo "Installing chaincode..."
                export CORE_PEER_LOCALMSPID="Org1MSP"
                export CORE_PEER_TLS_ROOTCERT_FILE=${PEER0_ORG1_TLSCERT}
                export CORE_PEER_MSPCONFIGPATH=${WORKSPACE}/test-network/organizations/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp
                export CORE_PEER_ADDRESS=${CORE_PEER_ADDRESS}

                peer lifecycle chaincode install ${CHAINCODE_NAME}.tar.gz
                '''
            }
        }

        // Other stages remain unchanged...
    }

    post {
        success {
            echo 'Pipeline execution succeeded!'
        }
        failure {
            echo 'Pipeline execution failed.'
        }
    }
}
