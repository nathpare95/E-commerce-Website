# E-commerce-Website
E-commerce Website
// Require the necessary modules
const express = require('express');
const app = express();
const bodyParser = require('body-parser');
const mongoose = require('mongoose');
const stripe = require('stripe')('your_stripe_secret_key_here');

// Connect to the database
mongoose.connect('mongodb://localhost/e-commerce-website', {
  useNewUrlParser: true,
  useUnifiedTopology: true,
  useCreateIndex: true,
});

// Define the product schema
const productSchema = new mongoose.Schema({
  name: { type: String, required: true },
  price: { type: Number, required: true },
  description: { type: String, required: true },
  image: { type: String, required: true },
  category: { type: String, required: true },
});

// Define the order schema
const orderSchema = new mongoose.Schema({
  items: [{ type: mongoose.Schema.Types.ObjectId, ref: 'Product' }],
  total: { type: Number, required: true },
  currency: { type: String, default: 'usd' },
  paymentId: { type: String },
  shippingAddress: { type: String, required: true },
  createdAt: { type: Date, default: Date.now },
});

// Define the product model
const Product = mongoose.model('Product', productSchema);

// Define the order model
const Order = mongoose.model('Order', orderSchema);

// Serve the web app
app.use(express.static('public'));
app.use(bodyParser.json());

// Get all products
app.get('/api/products', async (req, res) => {
  const products = await Product.find();
  res.json(products);
});

// Get a single product by ID
app.get('/api/products/:id', async (req, res) => {
  const product = await Product.findById(req.params.id);
  if (!product) {
    return res.status(404).json({ message: 'Product not found' });
  }
  res.json(product);
});

// Create a new order
app.post('/api/orders', async (req, res) => {
  const items = req.body.items;
  const total = items.reduce((sum, item) => sum + item.price, 0);
  const order = new Order({
    items: items.map(item => item._id),
    total,
    shippingAddress: req.body.shippingAddress,
  });
  try {
    const charge = await stripe.charges.create({
      amount: total * 100,
      currency: 'usd',
      source: req.body.token.id,
      description: 'E-commerce website order',
    });
    order.paymentId = charge.id;
    await order.save();
    res.status(201).json({ message: 'Order created' });
  } catch {
    res.status(500).json({ message: 'Error creating order' });
  }
});

// Start the server
app.listen(3000, () => {
  console.log('Server started on port 3000');
});
