// index.js - minimal backend for mobile testing
const express = require('express');
const bodyParser = require('body-parser');
const { createClient } = require('@supabase/supabase-js');

const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_KEY = process.env.SUPABASE_KEY;
const supabase = createClient(SUPABASE_URL, SUPABASE_KEY);

const app = express();
app.use(bodyParser.json());

// register
app.post('/register', async (req, res) => {
  const { email } = req.body;
  if (!email) return res.status(400).json({ error: 'email required' });

  const { data, error } = await supabase
    .from('users')
    .insert({ email })
    .select('*')
    .single();

  if (error) return res.status(400).json({ error: error.message });
  res.json({ user: data });
});

// deposit address
app.post('/deposit-address', async (req, res) => {
  const { user_id, asset_code } = req.body;
  if (!user_id || !asset_code) {
    return res.status(400).json({ error: 'user_id & asset_code required' });
  }

  const fakeAddress = 'ADDR_' + Math.random().toString(36).slice(2, 12);

  const { data, error } = await supabase
    .from('addresses')
    .insert({ user_id, asset_code, address: fakeAddress })
    .select('*')
    .single();

  if (error) return res.status(400).json({ error: error.message });
  res.json({ address: data });
});

// simulate deposit
app.post('/simulate-deposit', async (req, res) => {
  const { to_address, amount, asset_code } = req.body;
  if (!to_address || !amount || !asset_code) {
    return res.status(400).json({ error: 'missing fields' });
  }

  const tx = await supabase
    .from('onchain_transactions')
    .insert({
      tx_hash: 'TX_' + Date.now(),
      asset_code,
      to_address,
      amount,
      confirmations: 1,
      status: 'confirmed'
    })
    .select('*')
    .single();

  const addr = await supabase
    .from('addresses')
    .select('*')
    .eq('address', to_address)
    .single();

  if (addr.error || !addr.data) return res.status(400).json({ error: 'address not found' });

  const credit = await supabase
    .from('ledger')
    .insert({
      user_id: addr.data.user_id,
      asset_code,
      type: 'deposit',
      amount,
      balance_after: amount,
      status: 'completed',
      reference_id: tx.data.tx_hash
    })
    .select('*')
    .single();

  await supabase
    .from('onchain_transactions')
    .update({ linked_ledger_id: credit.data.id })
    .eq('id', tx.data.id);

  res.json({ ok: true, ledger: credit.data });
});

// wallets
app.get('/wallets/:user_id', async (req, res) => {
  const user_id = req.params.user_id;
  const { data, error } = await supabase
    .from('ledger')
    .select('*')
    .eq('user_id', user_id)
    .order('created_at', { ascending: false });

  if (error) return res.status(400).json({ error: error.message });
  res.json({ ledger: data });
});

// provably fair game
const crypto = require('crypto');
let currentServerSeed = null;
let currentServerHash = null;

app.get('/game/new-round', (req, res) => {
  currentServerSeed = crypto.randomBytes(32).toString('hex');
  currentServerHash = crypto.createHash('sha256').update(currentServerSeed).digest('hex');
  res.json({ server_hash: currentServerHash, round_id: Date.now().toString() });
});

app.post('/game/resolve', (req, res) => {
  const { client_seed = 'client', nonce = 0 } = req.body;
  if (!currentServerSeed) return res.status(400).json({ error: 'no round active' });

  const hmac = crypto.createHmac('sha256', currentServerSeed);
  hmac.update(`${client_seed}:${nonce}`);
  const hash = hmac.digest('hex');

  const int = parseInt(hash.slice(0, 12), 16);
  const max = Math.pow(2, 48) - 1;
  const rnd = int / max;
  const multiplier = +(1 + rnd * 10).toFixed(2);

  const revealedSeed = currentServerSeed;

  currentServerSeed = null;
  currentServerHash = null;

  res.json({
    multiplier,
    server_seed: revealedSeed,
    proof_hash: crypto.createHash('sha256').update(revealedSeed).digest('hex')
  });
});

const port = process.env.PORT || 10000;
app.listen(port, () => console.log('Server running on', port));
