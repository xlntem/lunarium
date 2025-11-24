# lunarium
lunarium/ # root monorepo
├─ package.json # workspace config (pnpm/npm/yarn)
├─ README.md
├─ packages/
│ ├─ shared-core/
│ │ ├─ package.json
│ │ └─ src/
│ │ ├─ crypto.ts
│ │ ├─ tx.ts
│ │ └─ index.ts
│ └─ daemon-manager/
│ ├─ package.json
│ └─ src/
│ ├─ server.ts
│ ├─ mnManager.ts
│ └─ routes.ts
├─ apps/
│ ├─ mobile/ # React Native (expo or bare) app
│ │ ├─ package.json
│ │ └─ App.tsx
│ │ └─ src/screens/Onboarding.tsx
│ └─ desktop/ # Electron + React app
│ ├─ package.json
│ ├─ main.ts # Electron main
│ └─ renderer/ # React renderer
│ └─ src/App.tsx
└─ docs/
└─ architecture.md
{
"name": "lunarium",
"private": true,
"version": "0.1.0",
"workspaces": [
"packages/*",
"apps/*"
],
"scripts": {
"bootstrap": "pnpm install",
"build": "pnpm -w run build",
"start:mobile": "pnpm --filter @lunarium/mobile start",
"start:desktop": "pnpm --filter @lunarium/desktop start",
"start:daemon": "pnpm --filter @lunarium/daemon-manager start"
}
}
package.json
{
"name": "@lunarium/shared-core",
"version": "0.1.0",
"main": "dist/index.js",
"types": "dist/index.d.ts",
"scripts": {
"build": "tsc",
"test": "jest"
},
"dependencies": {
"bip39": "^3.0.4",
"@noble/secp256k1": "^1.8.1",
"@noble/ed25519": "^1.7.1",
"tweetnacl": "^1.0.3",
"bs58": "^4.0.1"
},
"devDependencies": {
"typescript": "^4.9.5",
"jest": "^29.0.0",
"@types/jest": "^29.0.0"
}
}
src/crypto.ts
// packages/shared-core/src/crypto.ts
import * as bip39 from 'bip39';
import { utils as secpUtils, getPublicKey, sign } from '@noble/secp256k1';
import { randomBytes } from 'crypto';


export function generateMnemonic(strength = 256): string {
return bip39.generateMnemonic(strength);
}


export async function mnemonicToSeed(mnemonic: string, passphrase = ''): Promise<Buffer> {
const seed = await bip39.mnemonicToSeed(mnemonic, passphrase);
return seed;
}


export function derivePrivateKeyFromSeed(seed: Buffer, index = 0): Uint8Array {
// SIMPLE derivation for demo purposes: HMAC-SHA512 or HKDF recommended
// This is NOT BIP32 compliant — adapt to BIP32/BIP44 if using secp256k1 hierarchical wallets
const hash = secpUtils.sha256(seed.toString('hex') + index.toString());
// noble returns Promise for sha256; wrap in sync-friendly form
const sk = secpUtils.hexToBytes(Buffer.from(hash).toString('hex'));
return sk;
}


export async function getKeypairFromMnemonic(mnemonic: string, index = 0) {
const seed = await mnemonicToSeed(mnemonic);
// derive a private key (demo only)
const sk = secpUtils.sha256(seed);
const priv = secpUtils.bytesToHex(await sk);
const pub = getPublicKey(priv);
return { privKeyHex: priv, pubKeyHex: Buffer.from(pub).toString('hex') };
}

src/tx.ts
// packages/shared-core/src/tx.ts
import { sign } from '@noble/secp256k1';


export interface UnsignedTx {
from: string;
to: string;
value: string; // in smallest unit
nonce: number;
gasPrice?: string;
data?: string;
}


export async function signTransaction(unsignedTx: UnsignedTx, privKeyHex: string): Promise<string> {
// serializar tx (JSON canonical) -> hash -> assinar
const serialized = JSON.stringify(unsignedTx);
const msgHash = await import('crypto').then(c => c.createHash('sha256').update(serialized).digest('hex'));
const signature = await sign(msgHash, privKeyHex);
return signature; // adaptar para retornar rawTx conforme Lunarium
}
src/index.ts
export * from './crypto';
export * from './tx';
package.json
{
"name": "@lunarium/mobile",
"version": "0.1.0",
"private": true,
"main": "node_modules/expo/AppEntry.js",
"scripts": {
"start": "expo start",
"android": "expo start --android",
"ios": "expo start --ios"
},
"dependencies": {
"expo": "^48.0.0",
"react": "18.2.0",
"react-native": "0.71.0",
"@lunarium/shared-core": "*",
"react-native-keychain": "^8.1.1"
}
}
src/screens/Onboarding.tsx
// apps/mobile/src/screens/Onboarding.tsx
import React, { useState } from 'react';
import { View, Text, Button, TextInput, Alert } from 'react-native';
importsrc/screens/Onboarding.tsx


{ generateMnemonic, mnemonicToSeed } from '@lunarium/shared-core';
import * as Keychain from 'react-native-keychain';


export default function Onboarding({ navigation }: any) {
const [mnemonic, setMnemonic] = useState<string | null>(null);
const [password, setPassword] = useState('');


const createWallet = async () => {
const m = generateMnemonic();
setMnemonic(m);
// store securely using Keychain
await Keychain.setGenericPassword('mnemonic', m, { service: 'lunarium-mnemonic' });
Alert.alert('Wallet created', 'Anote sua seed em um local seguro.');
// navigate to app dashboard
navigation.replace('Dashboard');
};


const restoreWallet = async () => {
if (!password) return Alert.alert('Senha', 'Informe a senha de backup');
// demo: try to read mnemonic from keychain
const creds = await Keychain.getGenericPassword({ service: 'lunarium-mnemonic' });
if (creds) {
Alert.alert('Restored', 'Carteira restaurada (demo).');
navigation.replace('Dashboard');
} else {
Alert.alert('Erro', 'Nenhuma carteira encontrada.');
}
};


return (
<View style={{ flex: 1, padding: 20, justifyContent: 'center' }}>
<Text style={{ fontSize: 24, marginBottom: 20 }}>Bem-vindo à Lunarium</Text>
<Button title="Criar nova carteira" onPress={createWallet} />
<View style={{ height: 20 }} />
<TextInput
placeholder="Senha de backup (opcional)"
value={password}
onChangeText={setPassword}
secureTextEntry
/>
<Button title="Restaurar carteira" onPress={restoreWallet} />
{mnemonic ? (
<View style={{ marginTop: 20 }}>
<Text style={{ fontWeight: 'bold' }}>Seed (anote):</Text>
<Text selectable>{mnemonic}</Text>
</View>
) : null}
</View>
);
}
apps/mobile/App.tsx
import React from 'react';
import { NavigationContainer } from '@react-navigation/native';
import { createNativeStackNavigator } from '@react-navigation/native-stack';
import Onboarding from './src/screens/Onboarding';


const Stack = createNativeStackNavigator();


export default function App() {
return (
<NavigationContainer>
<Stack.Navigator screenOptions={{ headerShown: false }}>
<Stack.Screen name="Onboarding" component={Onboarding} />
<Stack.Screen name="Dashboard" component={() => <></>} />
</Stack.Navigator>
</NavigationContainer>
);
}
package.json (desktop)
{
"name": "@lunarium/desktop",
"version": "0.1.0",
"private": true,
"main": "dist/main.js",
"scripts": {
"start": "electron .",
"build": "tsc"
},
"dependencies": {
"electron": "^25.0.0",
"react": "18.2.0",
"react-dom": "18.2.0",
"@lunarium/shared-core": "*"
}
}
main.ts (Electron main process)
import { app, BrowserWindow } from 'electron';
import * as path from 'path';


function createWindow() {
const win = new BrowserWindow({
width: 1024,
height: 768,
webPreferences: {
preload: path.join(__dirname, 'preload.js'),
nodeIntegration: false,
contextIsolation: true
}
});
win.loadURL('http://localhost:3000'); // run react dev server
}


app.whenReady().then(createWindow);


app.on('window-all-closed', () => {
if (process.platform !== 'darwin') app.quit();
});
renderer/src/App.tsx basic placeholder
import React from 'react';


export default function App() {
return (
<div style={{ padding: 20 }}>
<h1>Lunarium Desktop</h1>
<p>Dashboard placeholder</p>
</div>
);
}
package.json
{
"name": "@lunarium/daemon-manager",
"version": "0.1.0",
"main": "dist/server.js",
"scripts": {
"start": "ts-node src/server.ts",
"build": "tsc"
},
"dependencies": {
"express": "^4.18.2",
"body-parser": "^1.20.2",
"node-fetch": "^3.3.2",
"uuid": "^9.0.0"
},
"devDependencies": {
"ts-node": "^10.9.1",
"typescript": "^4.9.5"
}
}
src/server.ts
import express from 'express';
import bodyParser from 'body-parser';
import { registerRoutes } from './routes';


const app = express();
app.use(bodyParser.json());


registerRoutes(app);


const PORT = process.env.PORT || 8081;
app.listen(PORT, () => console.log(`Daemon manager listening on ${PORT}`));
src/routes.ts
import { Application, Request, Response } from 'express';
import { MnManager } from './mnManager';


const manager = new MnManager();


export function registerRoutes(app: Application) {
app.post('/masternode/register', async (req: Request, res: Response) => {
try {
const { ip, port, operatorKey, collateralTx } = req.body;
const mn = await manager.register({ ip, port, operatorKey, collateralTx });
res.json({ success: true, mnId: mn.id });
} catch (err: any) {
res.status(500).json({ success: false, error: err.message });
}
});


app.get('/masternode/:id/status', async (req: Request, res: Response) => {
const id = req.params.id;
const status = manager.getStatus(id);
res.json({ success: true, status });
});


app.post('/masternode/:id/heartbeat', async (req: Request, res: Response) => {
const id = req.params.id;
manager.heartbeat(id);
res.json({ success: true });
});
}
`src/mnManage
