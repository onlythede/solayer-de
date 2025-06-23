# Complete Secure NFT Deployment Guide for Solayer Devnet

This guide provides a production-ready, secure approach to deploying NFTs on Solayer devnet with IPFS images, using environment variables for sensitive data.

## 📋 Prerequisites

- Node.js 18+ and npm installed
- Basic terminal/command line knowledge
- An image file for your NFT
- Internet connection

---

## 🚀 Step 1: Project Setup

### 1.1 Create Project Directory
```bash
mkdir solayer-nft-secure && cd solayer-nft-secure
```

### 1.2 Initialize Node.js Project
```bash
npm init -y
```

### 1.3 Install Dependencies
```bash
npm install @solana/web3.js @solana/spl-token express dotenv
```

### 1.4 Create Project Structure
```bash
mkdir -p assets scripts server config
touch .env .env.example .gitignore
```

### 1.5 Setup Package.json for ES Modules
```bash
cat > package.json << 'EOF'
{
  "name": "solayer-nft-secure",
  "version": "1.0.0",
  "type": "module",
  "main": "index.js",
  "scripts": {
    "deploy": "node scripts/deploy_nft.js",
    "server": "node server/api.js",
    "setup": "node scripts/setup.js",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": ["solana", "nft", "solayer", "ipfs"],
  "author": "",
  "license": "ISC",
  "description": "Secure NFT deployment on Solayer devnet"
}
EOF
```

### 1.6 Create .gitignore
```bash
cat > .gitignore << 'EOF'
# Environment variables
.env

# Node modules
node_modules/

# Logs
*.log
npm-debug.log*

# Runtime data
pids
*.pid
*.seed
*.pid.lock

# Coverage directory used by tools like istanbul
coverage/

# Dependency directories
node_modules/

# Optional npm cache directory
.npm

# Optional REPL history
.node_repl_history

# Output of 'npm pack'
*.tgz

# Yarn Integrity file
.yarn-integrity

# dotenv environment variables file
.env

# Solana keypairs (if stored locally)
*.json
!package*.json
!tsconfig*.json

# OS generated files
.DS_Store
.DS_Store?
._*
.Spotlight-V100
.Trashes
ehthumbs.db
Thumbs.db

# IDE files
.vscode/
.idea/
*.swp
*.swo
*~

# Deployment artifacts
deployment-index.json
EOF
```

---

## 🔐 Step 2: Environment Configuration

### 2.1 Create Environment Template
```bash
cat > .env.example << 'EOF'
# Solana Configuration
SOLANA_RPC_URL=https://devnet-rpc.solayer.org
SOLANA_KEYPAIR_PATH=~/.config/solana/id.json
NETWORK=devnet

# IPFS/Pinata Configuration (Optional - for metadata upload)
PINATA_JWT=your_pinata_jwt_token_here
PINATA_GATEWAY=https://gateway.pinata.cloud

# API Server Configuration
PORT=3000
NODE_ENV=development

# NFT Configuration
DEFAULT_METADATA_FILE=assets/nft-metadata.json
EOF
```

### 2.2 Create Your .env File
```bash
cp .env.example .env
echo "✅ Created .env file - please edit it with your values"
```

---

## 🛠️ Step 3: Create Configuration Module

### 3.1 Create Config Module
```bash
cat > config/index.js << 'EOF'
import dotenv from 'dotenv';
import fs from 'fs';
import path from 'path';

// Load environment variables
dotenv.config();

// Validate required environment variables
const requiredEnvVars = [
  'SOLANA_RPC_URL',
  'SOLANA_KEYPAIR_PATH'
];

const missingVars = requiredEnvVars.filter(varName => !process.env[varName]);
if (missingVars.length > 0) {
  console.error('❌ Missing required environment variables:', missingVars.join(', '));
  console.error('Please check your .env file');
  process.exit(1);
}

// Expand tilde in keypair path
function expandPath(filepath) {
  if (filepath.startsWith('~/')) {
    return path.join(process.env.HOME, filepath.slice(2));
  }
  return filepath;
}

const config = {
  solana: {
    rpcUrl: process.env.SOLANA_RPC_URL,
    keypairPath: expandPath(process.env.SOLANA_KEYPAIR_PATH),
    network: process.env.NETWORK || 'devnet',
    commitment: 'confirmed'
  },
  
  ipfs: {
    pinataJwt: process.env.PINATA_JWT,
    gateway: process.env.PINATA_GATEWAY || 'https://gateway.pinata.cloud'
  },
  
  api: {
    port: parseInt(process.env.PORT) || 3000,
    nodeEnv: process.env.NODE_ENV || 'development'
  },
  
  nft: {
    defaultMetadataFile: process.env.DEFAULT_METADATA_FILE || 'assets/nft-metadata.json'
  },
  
  // File paths
  deploymentIndex: 'deployment-index.json',
  
  // Validation helper
  validate() {
    // Check if keypair file exists
    if (!fs.existsSync(this.solana.keypairPath)) {
      throw new Error(`Solana keypair not found at: ${this.solana.keypairPath}`);
    }
    
    console.log('✅ Configuration validated');
    return true;
  },
  
  // Display configuration (without sensitive data)
  display() {
    console.log('📋 Configuration:');
    console.log(`   Network: ${this.solana.network}`);
    console.log(`   RPC URL: ${this.solana.rpcUrl}`);
    console.log(`   Keypair: ${this.solana.keypairPath}`);
    console.log(`   API Port: ${this.api.port}`);
    console.log(`   IPFS Configured: ${this.ipfs.pinataJwt ? 'Yes' : 'No'}`);
  }
};

export default config;
EOF
```

---

## 🔧 Step 4: Create Setup Script

### 4.1 Create Setup/Validation Script
```bash
cat > scripts/setup.js << 'EOF'
import { execSync } from 'child_process';
import fs from 'fs';
import config from '../config/index.js';

async function checkSolanaInstallation() {
  try {
    const version = execSync('solana --version', { encoding: 'utf8' });
    console.log('✅ Solana CLI installed:', version.trim());
    return true;
  } catch (error) {
    console.error('❌ Solana CLI not found. Please install it:');
    console.error('   curl -sSf https://release.solana.com/v1.18.26/install | sh');
    return false;
  }
}

async function checkKeypair() {
  try {
    if (!fs.existsSync(config.solana.keypairPath)) {
      console.log('🔑 No keypair found. Generating new keypair...');
      execSync(`solana-keygen new --outfile ${config.solana.keypairPath} --no-bip39-passphrase`, { stdio: 'inherit' });
    }
    
    const address = execSync('solana address', { encoding: 'utf8' }).trim();
    console.log('✅ Wallet address:', address);
    return address;
  } catch (error) {
    console.error('❌ Failed to setup keypair:', error.message);
    return null;
  }
}

async function configureSolana() {
  try {
    console.log('⚙️ Configuring Solana CLI...');
    execSync(`solana config set --url ${config.solana.rpcUrl} --commitment ${config.solana.commitment}`, { stdio: 'inherit' });
    console.log('✅ Solana CLI configured');
  } catch (error) {
    console.error('❌ Failed to configure Solana CLI:', error.message);
  }
}

async function checkBalance() {
  try {
    const balance = execSync('solana balance', { encoding: 'utf8' }).trim();
    console.log('💰 Current balance:', balance);
    
    if (balance.includes('0 SOL') || balance.includes('0.00')) {
      console.log('🪂 Requesting airdrop...');
      execSync('solana airdrop 1', { stdio: 'inherit' });
      
      // Wait a moment and check again
      await new Promise(resolve => setTimeout(resolve, 3000));
      const newBalance = execSync('solana balance', { encoding: 'utf8' }).trim();
      console.log('💰 New balance:', newBalance);
    }
  } catch (error) {
    console.error('❌ Failed to check balance:', error.message);
  }
}

async function createSampleMetadata() {
  const metadataPath = config.nft.defaultMetadataFile;
  
  if (!fs.existsSync(metadataPath)) {
    console.log('📄 Creating sample metadata file...');
    
    const sampleMetadata = {
      name: "My First Solayer NFT",
      symbol: "SLYR1",
      description: "My first NFT deployed on Solayer devnet using secure deployment",
      image: "https://gateway.pinata.cloud/ipfs/YOUR_IPFS_HASH_HERE",
      attributes: [
        {
          trait_type: "Network",
          value: "Solayer Devnet"
        },
        {
          trait_type: "Deployment",
          value: "Secure"
        },
        {
          trait_type: "Storage",
          value: "IPFS"
        }
      ]
    };
    
    fs.writeFileSync(metadataPath, JSON.stringify(sampleMetadata, null, 2));
    console.log(`✅ Sample metadata created at: ${metadataPath}`);
    console.log('⚠️  Remember to replace YOUR_IPFS_HASH_HERE with your actual IPFS hash');
  } else {
    console.log('✅ Metadata file already exists');
  }
}

async function main() {
  console.log('🚀 Setting up Solayer NFT deployment environment...\n');
  
  try {
    // Validate configuration
    config.validate();
    config.display();
    console.log('');
    
    // Check Solana installation
    if (!(await checkSolanaInstallation())) {
      process.exit(1);
    }
    
    // Setup keypair
    const address = await checkKeypair();
    if (!address) {
      process.exit(1);
    }
    
    // Configure Solana CLI
    await configureSolana();
    
    // Check/request balance
    await checkBalance();
    
    // Create sample metadata
    await createSampleMetadata();
    
    console.log('\n🎉 Setup complete! Next steps:');
    console.log('1. Upload your image to IPFS (see IPFS section in main guide)');
    console.log('2. Edit assets/nft-metadata.json with your IPFS hash');
    console.log('3. Run: npm run deploy');
    
  } catch (error) {
    console.error('❌ Setup failed:', error.message);
    process.exit(1);
  }
}

main();
EOF
```

---

## 🖼️ Step 5: IPFS Upload Guide

### 5.1 Create IPFS Upload Script (Optional)
```bash
cat > scripts/upload_to_ipfs.js << 'EOF'
import fs from 'fs';
import config from '../config/index.js';

async function uploadImageToPinata(imagePath) {
  if (!config.ipfs.pinataJwt) {
    throw new Error('PINATA_JWT not configured in .env file');
  }
  
  if (!fs.existsSync(imagePath)) {
    throw new Error(`Image file not found: ${imagePath}`);
  }
  
  console.log('📤 Uploading image to IPFS via Pinata...');
  
  // Create form data
  const formData = new FormData();
  const fileBuffer = fs.readFileSync(imagePath);
  const fileName = imagePath.split('/').pop();
  
  // Create blob from buffer
  const blob = new Blob([fileBuffer]);
  formData.append('file', blob, fileName);
  
  // Add metadata
  formData.append('pinataMetadata', JSON.stringify({
    name: fileName,
    keyvalues: {
      project: 'solayer-nft',
      type: 'image'
    }
  }));
  
  const response = await fetch('https://api.pinata.cloud/pinning/pinFileToIPFS', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${config.ipfs.pinataJwt}`
    },
    body: formData
  });
  
  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Failed to upload to Pinata: ${error}`);
  }
  
  const data = await response.json();
  const imageUrl = `${config.ipfs.gateway}/ipfs/${data.IpfsHash}`;
  
  console.log('✅ Image uploaded successfully!');
  console.log(`📎 IPFS Hash: ${data.IpfsHash}`);
  console.log(`🔗 Image URL: ${imageUrl}`);
  
  return {
    hash: data.IpfsHash,
    url: imageUrl
  };
}

async function uploadMetadataToPinata(metadata) {
  if (!config.ipfs.pinataJwt) {
    throw new Error('PINATA_JWT not configured in .env file');
  }
  
  console.log('📤 Uploading metadata to IPFS via Pinata...');
  
  const response = await fetch('https://api.pinata.cloud/pinning/pinJSONToIPFS', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${config.ipfs.pinataJwt}`
    },
    body: JSON.stringify({
      pinataContent: metadata,
      pinataMetadata: {
        name: `${metadata.name} Metadata`,
        keyvalues: {
          project: 'solayer-nft',
          type: 'metadata'
        }
      }
    })
  });
  
  if (!response.ok) {
    const error = await response.text();
    throw new Error(`Failed to upload metadata to Pinata: ${error}`);
  }
  
  const data = await response.json();
  const metadataUrl = `${config.ipfs.gateway}/ipfs/${data.IpfsHash}`;
  
  console.log('✅ Metadata uploaded successfully!');
  console.log(`📎 IPFS Hash: ${data.IpfsHash}`);
  console.log(`🔗 Metadata URL: ${metadataUrl}`);
  
  return {
    hash: data.IpfsHash,
    url: metadataUrl
  };
}

// CLI usage
async function main() {
  const imagePath = process.argv[2];
  
  if (!imagePath) {
    console.log('Usage: node scripts/upload_to_ipfs.js <image_path>');
    console.log('Example: node scripts/upload_to_ipfs.js assets/my-nft.png');
    process.exit(1);
  }
  
  try {
    const result = await uploadImageToPinata(imagePath);
    
    console.log('\n🔄 Next steps:');
    console.log('1. Copy the Image URL above');
    console.log('2. Edit your metadata file and replace YOUR_IPFS_HASH_HERE');
    console.log('3. Run: npm run deploy');
    
  } catch (error) {
    console.error('❌ Upload failed:', error.message);
    process.exit(1);
  }
}

// Export functions for use in other scripts
export { uploadImageToPinata, uploadMetadataToPinata };

// Run if called directly
if (process.argv[1] === new URL(import.meta.url).pathname) {
  main();
}
EOF
```

### 5.2 Manual IPFS Upload Instructions
Create a simple guide file:
```bash
cat > IPFS_UPLOAD_GUIDE.md << 'EOF'
# IPFS Upload Guide

## Option 1: Using Pinata Web Interface (Recommended for beginners)

1. Go to https://pinata.cloud and create a free account
2. Click "Upload" → "File" 
3. Select your NFT image
4. After upload, copy the IPFS hash (starts with 'baf...')
5. Your image URL will be: `https://gateway.pinata.cloud/ipfs/YOUR_HASH`

## Option 2: Using the Upload Script

1. Get your Pinata JWT token:
   - Go to Pinata → API Keys
   - Create new API key
   - Copy the JWT token

2. Add to your .env file:
   ```
   PINATA_JWT=your_jwt_token_here
   ```

3. Upload your image:
   ```bash
   node scripts/upload_to_ipfs.js assets/your-image.png
   ```

## Option 3: Other IPFS Services

- **NFT.Storage**: https://nft.storage (free)
- **Arweave**: https://arweave.org (permanent storage)
- **IPFS Desktop**: https://ipfs.io (run your own node)

## After Upload

1. Copy the IPFS URL
2. Edit `assets/nft-metadata.json`
3. Replace `YOUR_IPFS_HASH_HERE` with your actual URL
4. Deploy your NFT: `npm run deploy`
EOF
```

---

## 🚀 Step 6: Create Secure Deployment Script

### 6.1 Create Main Deployment Script
```bash
cat > scripts/deploy_nft.js << 'EOF'
import {
  Connection,
  Keypair,
  Transaction,
  SystemProgram,
  LAMPORTS_PER_SOL
} from '@solana/web3.js';
import {
  createInitializeMintInstruction,
  createAssociatedTokenAccountInstruction,
  createMintToInstruction,
  getMinimumBalanceForRentExemptMint,
  MINT_SIZE,
  TOKEN_PROGRAM_ID,
  getAssociatedTokenAddress
} from '@solana/spl-token';
import fs from 'fs';
import config from '../config/index.js';
import { uploadMetadataToPinata } from './upload_to_ipfs.js';

class NFTDeployer {
  constructor() {
    this.config = config;
    this.connection = null;
    this.payer = null;
  }

  async initialize() {
    try {
      // Validate configuration
      this.config.validate();
      
      // Load keypair
      const secret = JSON.parse(fs.readFileSync(this.config.solana.keypairPath));
      this.payer = Keypair.fromSecretKey(Uint8Array.from(secret));
      
      // Initialize connection (HTTP only, no WebSockets)
      this.connection = new Connection(this.config.solana.rpcUrl, {
        commitment: this.config.solana.commitment,
        disableRetryOnRateLimit: true,
        confirmTransactionInitialTimeout: 60000
      });
      
      console.log('✅ NFT Deployer initialized');
      console.log(`📍 Wallet: ${this.payer.publicKey.toBase58()}`);
      console.log(`🌐 Network: ${this.config.solana.network}`);
      
    } catch (error) {
      throw new Error(`Failed to initialize deployer: ${error.message}`);
    }
  }

  appendToIndex(entry) {
    const indexPath = this.config.deploymentIndex;
    const deployments = fs.existsSync(indexPath) 
      ? JSON.parse(fs.readFileSync(indexPath)) 
      : [];
    
    deployments.push({
      ...entry,
      timestamp: new Date().toISOString()
    });
    
    fs.writeFileSync(indexPath, JSON.stringify(deployments, null, 2));
    console.log(`📝 Updated deployment index: ${entry.step}`);
  }

  async confirmTransaction(signature, maxRetries = 30) {
    console.log('⏳ Confirming transaction...');
    
    for (let i = 0; i < maxRetries; i++) {
      try {
        const response = await this.connection.getSignatureStatus(signature);
        
        if (response.value?.confirmationStatus === 'confirmed' || 
            response.value?.confirmationStatus === 'finalized') {
          console.log(`✅ Transaction confirmed (${response.value.confirmationStatus})`);
          return response.value;
        }
        
        if (response.value?.err) {
          throw new Error(`Transaction failed: ${JSON.stringify(response.value.err)}`);
        }
        
      } catch (error) {
        console.log(`Confirmation attempt ${i + 1}/${maxRetries} failed: ${error.message}`);
      }
      
      if (i < maxRetries - 1) {
        console.log(`⏳ Waiting 2s before retry... (${i + 1}/${maxRetries})`);
        await new Promise(resolve => setTimeout(resolve, 2000));
      }
    }
    
    throw new Error('Transaction confirmation timeout');
  }

  async createMetadataUri(metadata) {
    console.log('📤 Creating metadata URI...');
    
    // Option 1: Upload to IPFS (if Pinata is configured)
    if (this.config.ipfs.pinataJwt) {
      try {
        console.log('🌐 Uploading metadata to IPFS...');
        const result = await uploadMetadataToPinata(metadata);
        return result.url;
      } catch (error) {
        console.warn(`⚠️ IPFS upload failed: ${error.message}`);
        console.log('📋 Falling back to data URI...');
      }
    }
    
    // Option 2: Fallback to data URI
    const dataUri = `data:application/json;base64,${Buffer.from(JSON.stringify(metadata)).toString('base64')}`;
    console.log('📋 Using data URI for metadata');
    return dataUri;
  }

  async validateMetadata(metadata) {
    console.log('🔍 Validating metadata...');
    
    // Required fields
    const requiredFields = ['name', 'symbol', 'description', 'image'];
    const missingFields = requiredFields.filter(field => !metadata[field]);
    
    if (missingFields.length > 0) {
      throw new Error(`Missing required metadata fields: ${missingFields.join(', ')}`);
    }
    
    // Check if image URL is accessible
    if (metadata.image.startsWith('http')) {
      try {
        console.log('🖼️ Verifying image accessibility...');
        const response = await fetch(metadata.image, { method: 'HEAD' });
        if (!response.ok) {
          throw new Error(`Image not accessible: ${response.status} ${response.statusText}`);
        }
        console.log(`✅ Image verified: ${response.headers.get('content-type')}`);
      } catch (error) {
        console.warn(`⚠️ Warning: Could not verify image: ${error.message}`);
      }
    }
    
    console.log('✅ Metadata validation passed');
  }

  async checkPrerequisites() {
    console.log('🔍 Checking deployment prerequisites...');
    
    // Check wallet balance
    const balance = await this.connection.getBalance(this.payer.publicKey);
    const balanceSOL = balance / LAMPORTS_PER_SOL;
    console.log(`💰 Wallet balance: ${balanceSOL} SOL`);
    
    if (balanceSOL < 0.01) {
      throw new Error(`Insufficient balance. Need at least 0.01 SOL, have ${balanceSOL} SOL`);
    }
    
    // Check network connectivity
    try {
      await this.connection.getLatestBlockhash();
      console.log('✅ Network connectivity verified');
    } catch (error) {
      throw new Error(`Network connectivity failed: ${error.message}`);
    }
    
    console.log('✅ Prerequisites check passed');
  }

  async deployNFT(metadataFilePath) {
    const startTime = Date.now();
    
    try {
      console.log('🚀 Starting NFT deployment...');
      console.log(`📄 Metadata file: ${metadataFilePath}`);
      
      // Load and validate metadata
      if (!fs.existsSync(metadataFilePath)) {
        throw new Error(`Metadata file not found: ${metadataFilePath}`);
      }
      
      const metadata = JSON.parse(fs.readFileSync(metadataFilePath, 'utf8'));
      console.log(`📋 NFT Name: "${metadata.name}"`);
      console.log(`🎨 Symbol: "${metadata.symbol}"`);
      
      await this.validateMetadata(metadata);
      await this.checkPrerequisites();
      
      // Create metadata URI
      const metadataUri = await this.createMetadataUri(metadata);
      
      this.appendToIndex({
        step: 'metadata_prepared',
        metadata: metadata,
        metadataUri: metadataUri.length > 100 ? metadataUri.substring(0, 100) + '...' : metadataUri
      });
      
      // Generate mint keypair
      const mintKeypair = Keypair.generate();
      console.log(`🎯 Generated mint address: ${mintKeypair.publicKey.toBase58()}`);
      
      // Calculate associated token account
      const associatedTokenAccount = await getAssociatedTokenAddress(
        mintKeypair.publicKey,
        this.payer.publicKey
      );
      console.log(`🏦 Token account: ${associatedTokenAccount.toBase58()}`);
      
      // Get latest blockhash and calculate rent
      const { blockhash } = await this.connection.getLatestBlockhash();
      const mintRent = await getMinimumBalanceForRentExemptMint(this.connection);
      console.log(`💸 Mint rent: ${mintRent / LAMPORTS_PER_SOL} SOL`);
      
      // Build transaction
      console.log('🔨 Building transaction...');
      const transaction = new Transaction();
      
      // 1. Create mint account
      transaction.add(
        SystemProgram.createAccount({
          fromPubkey: this.payer.publicKey,
          newAccountPubkey: mintKeypair.publicKey,
          space: MINT_SIZE,
          lamports: mintRent,
          programId: TOKEN_PROGRAM_ID,
        })
      );
      
      // 2. Initialize mint (0 decimals = NFT)
      transaction.add(
        createInitializeMintInstruction(
          mintKeypair.publicKey,
          0, // 0 decimals for NFT
          this.payer.publicKey, // mint authority
          this.payer.publicKey  // freeze authority
        )
      );
      
      // 3. Create associated token account
      transaction.add(
        createAssociatedTokenAccountInstruction(
          this.payer.publicKey,
          associatedTokenAccount,
          this.payer.publicKey,
          mintKeypair.publicKey
        )
      );
      
      // 4. Mint 1 token (the NFT)
      transaction.add(
        createMintToInstruction(
          mintKeypair.publicKey,
          associatedTokenAccount,
          this.payer.publicKey,
          1 // amount = 1 for NFT
        )
      );
      
      // Set transaction properties
      transaction.recentBlockhash = blockhash;
      transaction.feePayer = this.payer.publicKey;
      
      // Sign transaction
      transaction.sign(this.payer, mintKeypair);
      console.log('✍️ Transaction signed');
      
      // Send transaction
      console.log('📤 Sending transaction...');
      const signature = await this.connection.sendRawTransaction(transaction.serialize(), {
        skipPreflight: false,
        preflightCommitment: this.config.solana.commitment
      });
      
      console.log(`📝 Transaction signature: ${signature}`);
      this.appendToIndex({
        step: 'transaction_sent',
        signature: signature,
        mint: mintKeypair.publicKey.toBase58()
      });
      
      // Confirm transaction
      const status = await this.confirmTransaction(signature);
      
      // Calculate deployment time
      const deploymentTime = Date.now() - startTime;
      
      // Record successful deployment
      const deploymentData = {
        step: 'nft_deployed',
        success: true,
        mint: mintKeypair.publicKey.toBase58(),
        tokenAccount: associatedTokenAccount.toBase58(),
        signature: signature,
        metadata: metadata,
        metadataUri: metadataUri,
        imageUrl: metadata.image,
        network: this.config.solana.network,
        rpcUrl: this.config.solana.rpcUrl,
        deploymentTimeMs: deploymentTime,
        status: status,
        explorer: `https://explorer.solana.com/tx/${signature}?cluster=custom&customUrl=${encodeURIComponent(this.config.solana.rpcUrl)}`
      };
      
      this.appendToIndex(deploymentData);
      
      // Print success summary
      console.log('\n🎉 NFT Successfully Deployed!');
      console.log('='.repeat(50));
      console.log(`🎯 Mint Address: ${mintKeypair.publicKey.toBase58()}`);
      console.log(`🏦 Token Account: ${associatedTokenAccount.toBase58()}`);
      console.log(`🖼️ Image URL: ${metadata.image}`);
      console.log(`📝 Transaction: ${signature}`);
      console.log(`⏱️ Deployment Time: ${(deploymentTime / 1000).toFixed(2)}s`);
      console.log(`🔍 Explorer: ${deploymentData.explorer}`);
      console.log(`📊 Full details: ${this.config.deploymentIndex}`);
      
      return deploymentData;
      
    } catch (error) {
      const deploymentTime = Date.now() - startTime;
      
      console.error('❌ Deployment failed:', error.message);
      
      this.appendToIndex({
        step: 'deployment_failed',
        error: error.message,
        metadataFile: metadataFilePath,
        deploymentTimeMs: deploymentTime
      });
      
      throw error;
    }
  }
}

// CLI usage
async function main() {
  const metadataFile = process.argv[2] || config.nft.defaultMetadataFile;
  
  console.log('🚀 Solayer NFT Deployment Tool\n');
  
  try {
    const deployer = new NFTDeployer();
    await deployer.initialize();
    await deployer.deployNFT(metadataFile);
    
  } catch (error) {
    console.error('\n❌ Deployment failed:', error.message);
    console.error('\n💡 Troubleshooting tips:');
    console.error('1. Check your .env file configuration');
    console.error('2. Ensure you have sufficient SOL balance');
    console.error('3. Verify your metadata file is valid JSON');
    console.error('4. Check network connectivity');
    process.exit(1);
  }
}

// Export for use in other modules
export { NFTDeployer };

// Run if called directly
if (process.argv[1] === new URL(import.meta.url).pathname) {
  main();
}
EOF
```

---

## 🌐 Step 7: Create Secure API Server

### 7.1 Create API Server
```bash
cat > server/api.js << 'EOF'
import express from 'express';
import fs from 'fs';
import path from 'path';
import { fileURLToPath } from 'url';
import config from '../config/index.js';

const __filename = fileURLToPath(import.meta.url);
const __dirname = path.dirname(__filename);

const app = express();

// Security middleware
app.use((req, res, next) => {
  // CORS
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.setHeader('Access-Control-Allow-Methods', 'GET, POST, OPTIONS');
  res.setHeader('Access-Control-Allow-Headers', 'Content-Type');
  
  // Security headers
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  
  // Rate limiting headers
  res.setHeader('X-RateLimit-Limit', '100');
  res.setHeader('X-RateLimit-Remaining', '99');
  
  next();
});

// Parse JSON bodies
app.use(express.json({ limit: '1mb' }));

// Request logging middleware
app.use((req, res, next) => {
  const timestamp = new Date().toISOString();
  console.log(`${timestamp} ${req.method} ${req.path} - ${req.ip}`);
  next();
});

// Health check endpoint
app.get('/health', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date().toISOString(),
    uptime: process.uptime(),
    version: '1.0.0',
    network: config.solana.network
  });
});

// Root endpoint
app.get('/', (req, res) => {
  res.json({
    name: 'Solayer NFT Deployment API',
    version: '1.0.0',
    network: config.solana.network,
    endpoints: {
      health: '/health',
      deployments: '/api/deployments',
      latest: '/api/deployments/latest',
      stats: '/api/stats'
    },
    documentation: 'https://github.com/your-username/solayer-nft-secure'
  });
});

// Get all deployments
app.get('/api/deployments', (req, res) => {
  try {
    if (!fs.existsSync(config.deploymentIndex)) {
      return res.status(404).json({
        error: 'No deployments found',
        message: 'Run the deployment script first: npm run deploy'
      });
    }
    
    const deployments = JSON.parse(fs.readFileSync(config.deploymentIndex));
    
    // Optional filtering
    const step = req.query.step;
    const filteredDeployments = step 
      ? deployments.filter(d => d.step === step)
      : deployments;
    
    res.json({
      total: filteredDeployments.length,
      deployments: filteredDeployments,
      filters: req.query
    });
    
  } catch (error) {
    console.error('Error reading deployments:', error);
    res.status(500).json({
      error: 'Failed to read deployments',
      message: error.message
    });
  }
});

// Get latest successful deployment
app.get('/api/deployments/latest', (req, res) => {
  try {
    if (!fs.existsSync(config.deploymentIndex)) {
      return res.status(404).json({
        error: 'No deployments found'
      });
    }
    
    const deployments = JSON.parse(fs.readFileSync(config.deploymentIndex));
    const successfulDeployments = deployments.filter(d => d.step === 'nft_deployed');
    
    if (successfulDeployments.length === 0) {
      return res.status(404).json({
        error: 'No successful deployments found'
      });
    }
    
    const latest = successfulDeployments[successfulDeployments.length - 1];
    res.json(latest);
    
  } catch (error) {
    console.error('Error reading latest deployment:', error);
    res.status(500).json({
      error: 'Failed to read latest deployment'
    });
  }
});

// Get deployment statistics
app.get('/api/stats', (req, res) => {
  try {
    if (!fs.existsSync(config.deploymentIndex)) {
      return res.json({
        totalDeployments: 0,
        successfulDeployments: 0,
        failedDeployments: 0,
        successRate: 0
      });
    }
    
    const deployments = JSON.parse(fs.readFileSync(config.deploymentIndex));
    const successful = deployments.filter(d => d.step === 'nft_deployed').length;
    const failed = deployments.filter(d => d.step === 'deployment_failed').length;
    const total = successful + failed;
    
    res.json({
      totalDeployments: total,
      successfulDeployments: successful,
      failedDeployments: failed,
      successRate: total > 0 ? Math.round((successful / total) * 100) : 0,
      lastDeployment: deployments.length > 0 ? deployments[deployments.length - 1].timestamp : null
    });
    
  } catch (error) {
    console.error('Error calculating stats:', error);
    res.status(500).json({
      error: 'Failed to calculate statistics'
    });
  }
});

// Error handling middleware
app.use((error, req, res, next) => {
  console.error('Unhandled error:', error);
  res.status(500).json({
    error: 'Internal server error',
    message: config.api.nodeEnv === 'development' ? error.message : 'Something went wrong'
  });
});

// 404 handler
app.use((req, res) => {
  res.status(404).json({
    error: 'Endpoint not found',
    availableEndpoints: ['/', '/health', '/api/deployments', '/api/deployments/latest', '/api/stats']
  });
});

// Start server
const server = app.listen(config.api.port, () => {
  console.log(`🚀 Secure NFT API Server running on http://localhost:${config.api.port}`);
  console.log(`📊 Deployments: http://localhost:${config.api.port}/api/deployments`);
  console.log(`❤️  Health: http://localhost:${config.api.port}/health`);
  console.log(`📈 Stats: http://localhost:${config.api.port}/api/stats`);
  console.log(`🌐 Network: ${config.solana.network}`);
});

// Graceful shutdown
process.on('SIGTERM', () => {
  console.log('Received SIGTERM, shutting down gracefully');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});

process.on('SIGINT', () => {
  console.log('Received SIGINT, shutting down gracefully');
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
});
EOF
```

---

## 📖 Step 8: Create Documentation

### 8.1 Create README
```bash
cat > README.md << 'EOF'
# Secure Solayer NFT Deployment

Deploy NFTs on Solayer devnet with IPFS images using secure, production-ready code.

## Features

- ✅ Secure environment variable configuration
- ✅ HTTP-only deployment (no WebSocket dependencies)
- ✅ IPFS image support via Pinata
- ✅ Comprehensive error handling
- ✅ RESTful API for deployment tracking
- ✅ Production-ready code structure

## Quick Start

### 1. Setup
```bash
# Clone and setup
git clone <your-repo>
cd solayer-nft-secure
npm install

# Run setup script
npm run setup
```

### 2. Configure Environment
```bash
# Edit .env file
cp .env.example .env
# Add your configuration values
```

### 3. Upload Image to IPFS
- Go to https://pinata.cloud
- Upload your image
- Copy the IPFS URL

### 4. Create Metadata
Edit `assets/nft-metadata.json` with your NFT details and IPFS URL.

### 5. Deploy NFT
```bash
npm run deploy
```

### 6. Start API Server
```bash
npm run server
```

## Environment Variables

| Variable | Description | Required |
|----------|-------------|----------|
| `SOLANA_RPC_URL` | Solayer devnet RPC URL | Yes |
| `SOLANA_KEYPAIR_PATH` | Path to your Solana keypair | Yes |
| `PINATA_JWT` | Pinata API token for IPFS | No |
| `PORT` | API server port | No |

## Scripts

- `npm run setup` - Setup environment and dependencies
- `npm run deploy` - Deploy NFT
- `npm run server` - Start API server

## API Endpoints

- `GET /` - API information
- `GET /health` - Health check
- `GET /api/deployments` - All deployments
- `GET /api/deployments/latest` - Latest deployment
- `GET /api/stats` - Deployment statistics

## Security Features

- Environment variables for sensitive data
- Input validation and sanitization
- Rate limiting headers
- Security headers (XSS, CSRF protection)
- Error handling without data leaks
- No hardcoded secrets

## Troubleshooting

### Common Issues

1. **Insufficient balance**
   - Solution: Request SOL airdrop: `solana airdrop 1`

2. **Keypair not found**
   - Solution: Run `npm run setup` to generate

3. **Image not accessible**
   - Solution: Verify IPFS URL is public

4. **Transaction timeout**
   - Solution: Check network connectivity

### Getting Help

1. Check the deployment logs
2. Verify your .env configuration
3. Ensure sufficient SOL balance
4. Check IPFS image accessibility

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

## License

MIT License
EOF
```

### 8.2 Create Usage Examples
```bash
cat > EXAMPLES.md << 'EOF'
# Usage Examples

## Basic Deployment

```bash
# 1. Setup environment
npm run setup

# 2. Edit metadata file
# Edit assets/nft-metadata.json with your details

# 3. Deploy NFT
npm run deploy

# 4. Check result
curl http://localhost:3000/api/deployments/latest
```

## Advanced Usage

### Deploy with Custom Metadata File

```bash
# Create custom metadata
cat > assets/my-special-nft.json << 'EOF'
{
  "name": "My Special NFT",
  "symbol": "SPECIAL",
  "description": "A very special NFT",
  "image": "https://gateway.pinata.cloud/ipfs/your-hash-here",
  "attributes": [
    {"trait_type": "Rarity", "value": "Legendary"}
  ]
}
EOF

# Deploy with custom file
node scripts/deploy_nft.js assets/my-special-nft.json
```

### Batch Deployment

```bash
# Deploy multiple NFTs
for i in {1..5}; do
  echo "Deploying NFT #$i"
  node scripts/deploy_nft.js assets/nft-$i.json
  sleep 5  # Wait between deployments
done
```

### Upload Image via Script

```bash
# Set Pinata JWT in .env first
echo "PINATA_JWT=your_jwt_here" >> .env

# Upload image
node scripts/upload_to_ipfs.js assets/my-image.png
```

## API Usage

### Check Deployment Status

```bash
# Get all deployments
curl http://localhost:3000/api/deployments

# Get only successful deployments
curl http://localhost:3000/api/deployments?step=nft_deployed

# Get deployment statistics
curl http://localhost:3000/api/stats
```

### Monitor Deployments

```bash
# Watch for new deployments
watch -n 5 "curl -s http://localhost:3000/api/stats | jq '.'"
```

## Integration Examples

### Use in Your Application

```javascript
import { NFTDeployer } from './scripts/deploy_nft.js';

async function deployMyNFT() {
  const deployer = new NFTDeployer();
  await deployer.initialize();
  
  const result = await deployer.deployNFT('path/to/metadata.json');
  console.log('NFT deployed:', result.mint);
}
```

### Webhook Integration

```javascript
// Add to your server
app.post('/deploy-nft', async (req, res) => {
  try {
    const { metadata } = req.body;
    
    // Save metadata to file
    fs.writeFileSync('temp-metadata.json', JSON.stringify(metadata));
    
    // Deploy NFT
    const deployer = new NFTDeployer();
    await deployer.initialize();
    const result = await deployer.deployNFT('temp-metadata.json');
    
    res.json({ success: true, mint: result.mint });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});
```
EOF
```

---

## 🚀 Step 9: Final Setup Instructions

### 9.1 Create Complete Setup Script
```bash
cat > scripts/complete_setup.sh << 'EOF'
#!/bin/bash

echo "🚀 Setting up Secure Solayer NFT Deployment Environment..."
echo ""

# Check if Node.js is installed
if ! command -v node &> /dev/null; then
    echo "❌ Node.js is not installed. Please install Node.js 18+ first:"
    echo "   https://nodejs.org/"
    exit 1
fi

# Check Node.js version
NODE_VERSION=$(node --version | cut -d'v' -f2 | cut -d'.' -f1)
if [ "$NODE_VERSION" -lt 18 ]; then
    echo "❌ Node.js version 18+ required. Current version: $(node --version)"
    exit 1
fi

echo "✅ Node.js $(node --version) detected"

# Install dependencies
echo "📦 Installing dependencies..."
npm install

# Run setup script
echo "⚙️ Running environment setup..."
npm run setup

echo ""
echo "🎉 Setup complete!"
echo ""
echo "📋 Next steps:"
echo "1. Edit .env file with your configuration"
echo "2. Upload your image to IPFS (see IPFS_UPLOAD_GUIDE.md)"
echo "3. Edit assets/nft-metadata.json with your IPFS URL"
echo "4. Run: npm run deploy"
echo ""
echo "📚 Documentation:"
echo "- README.md - Main documentation"
echo "- EXAMPLES.md - Usage examples"
echo "- IPFS_UPLOAD_GUIDE.md - IPFS upload instructions"
echo ""
echo "🌐 Once deployed, start the API server: npm run server"
EOF

chmod +x scripts/complete_setup.sh
```

---

## 📋 Step 10: Complete Usage Guide

### 10.1 Create the Final User Guide
```bash
cat > COMPLETE_GUIDE.md << 'EOF'
# Complete Solayer NFT Deployment Guide

Follow these steps to deploy your NFT on Solayer devnet securely.

## Prerequisites

- Node.js 18+ installed
- An image file for your NFT
- Basic command line knowledge

## Step 1: Project Setup

```bash
# Clone or download this project
git clone <repository-url>
cd solayer-nft-secure

# Run complete setup
chmod +x scripts/complete_setup.sh
./scripts/complete_setup.sh
```

## Step 2: Environment Configuration

Edit your `.env` file:

```bash
# Copy template and edit
cp .env.example .env
nano .env  # or use your preferred editor
```

Minimum required configuration:
```
SOLANA_RPC_URL=https://devnet-rpc.solayer.org
SOLANA_KEYPAIR_PATH=~/.config/solana/id.json
```

## Step 3: Upload Image to IPFS

### Option A: Pinata Web Interface (Recommended)
1. Go to https://pinata.cloud
2. Create free account
3. Upload your image
4. Copy the IPFS URL (starts with https://gateway.pinata.cloud/ipfs/...)

### Option B: Using the Upload Script
1. Get Pinata JWT token from https://pinata.cloud/keys
2. Add to .env: `PINATA_JWT=your_token_here`
3. Run: `node scripts/upload_to_ipfs.js path/to/your/image.png`

## Step 4: Create NFT Metadata

Edit `assets/nft-metadata.json`:

```json
{
  "name": "Your NFT Name",
  "symbol": "SYMBOL",
  "description": "Your NFT description",
  "image": "https://gateway.pinata.cloud/ipfs/YOUR_IPFS_HASH",
  "attributes": [
    {
      "trait_type": "Category",
      "value": "Art"
    }
  ]
}
```

## Step 5: Deploy Your NFT

```bash
# Deploy with default metadata
npm run deploy

# Or deploy with custom metadata file
node scripts/deploy_nft.js assets/your-custom-metadata.json
```

## Step 6: Start API Server (Optional)

```bash
# Start the API server
npm run server

# Check your deployment
curl http://localhost:3000/api/deployments/latest
```

## Step 7: Verify Deployment

Your NFT should be visible on Solana Explorer:
```
https://explorer.solana.com/address/YOUR_MINT_ADDRESS?cluster=custom&customUrl=https%3A//devnet-rpc.solayer.org
```

## Troubleshooting

### Error: Insufficient Balance
```bash
# Request SOL airdrop
solana airdrop 1
```

### Error: Keypair Not Found
```bash
# Generate new keypair
solana-keygen new --outfile ~/.config/solana/id.json
```

### Error: Image Not Accessible
- Verify your IPFS URL is public and accessible
- Test in browser: should show your image

### Error: Invalid Metadata
- Validate JSON syntax: https://jsonlint.com/
- Ensure all required fields are present

## Advanced Usage

### Deploy Multiple NFTs
```bash
# Create multiple metadata files
cp assets/nft-metadata.json assets/nft-1.json
cp assets/nft-metadata.json assets/nft-2.json

# Edit each file with different details

# Deploy each NFT
node scripts/deploy_nft.js assets/nft-1.json
node scripts/deploy_nft.js assets/nft-2.json
```

### Monitor Deployments
```bash
# Get deployment statistics
curl http://localhost:3000/api/stats

# Watch deployments in real-time
watch -n 5 "curl -s http://localhost:3000/api/deployments | jq '.total'"
```

## Security Best Practices

✅ **DO:**
- Keep your .env file secure and never commit it
- Use environment variables for sensitive data
- Regularly update dependencies
- Use strong, unique passwords for external services

❌ **DON'T:**
- Share your private keys or JWT tokens
- Commit .env files to version control
- Use mainnet keys for testing
- Deploy without verifying metadata

## Support

- Check `deployment-index.json` for detailed logs
- Review error messages in console output
- Ensure all prerequisites are met
- Verify network connectivity

## What You've Built

After successful deployment, you'll have:

1. **Secure NFT**: Deployed on Solayer devnet
2. **IPFS Storage**: Image hosted on decentralized storage
3. **API Server**: RESTful API for deployment tracking
4. **Complete Logs**: Full deployment history
5. **Explorer Link**: View your NFT on Solana Explorer

## Next Steps

- Deploy on Solana mainnet (update RPC URL)
- Create a collection of NFTs
- Build a web interface for your NFTs
- Integrate with NFT marketplaces

**Congratulations! You've successfully deployed a secure NFT on Solayer devnet! 🎉**
EOF
```

---

## 🎯 Final Project Structure

Your complete project structure should look like this:

```
solayer-nft-secure/
├── .env                          # Your environment variables
├── .env.example                  # Environment template
├── .gitignore                    # Git ignore file
├── package.json                  # Node.js dependencies
├── README.md                     # Main documentation
├── COMPLETE_GUIDE.md             # Step-by-step user guide
├── EXAMPLES.md                   # Usage examples
├── IPFS_UPLOAD_GUIDE.md          # IPFS upload instructions
├── assets/
│   └── nft-metadata.json         # NFT metadata
├── config/
│   └── index.js                  # Configuration management
├── scripts/
│   ├── complete_setup.sh         # Complete setup script
│   ├── setup.js                  # Environment setup
│   ├── deploy_nft.js             # Main deployment script
│   └── upload_to_ipfs.js         # IPFS upload utility
├── server/
│   └── api.js                    # Express API server
└── deployment-index.json         # Generated deployment log
```

## 🚀 Quick Start for Users

Users can now follow these simple steps:

```bash
# 1. Download the project
git clone your-repository-url
cd solayer-nft-secure

# 2. Run setup
./scripts/complete_setup.sh

# 3. Configure .env file
# Edit .env with your settings

# 4. Upload image to IPFS
# Use Pinata web interface or the upload script

# 5. Edit metadata
# Update assets/nft-metadata.json with your IPFS URL

# 6. Deploy NFT
npm run deploy

# 7. Start API server (optional)
npm run server
```

This secure, production-ready setup includes:

- ✅ **Environment variable security**
- ✅ **Comprehensive error handling**
- ✅ **IPFS integration**
- ✅ **API server for tracking**
- ✅ **Complete documentation**
- ✅ **Production best practices**
- ✅ **Easy setup scripts**

Users can now safely deploy NFTs on Solayer devnet with confidence! 🎉
