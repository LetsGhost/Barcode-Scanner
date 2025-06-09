<template>
<div>
  <h1>Upload a image of an barcode and get the Item!</h1>
  <input type="file" name="Image" id="upload" accept="image/*" ref="fileInput" />
  <button id="button" @click="onReadClick">READ IT!</button>
  <div id="result" class="result">
  </div>
  <button
  class="copy"
  @click="copyToClipboard(`${productName}, ${productBrand}`)"
  :disabled="!productName"
>
  Copy for MyFitnessPal
</button>
</div>
</template>

<script setup>
import { ref } from 'vue';
import Quagga from '@ericblade/quagga2';

const fileInput = ref(null);

const productName = ref('');
const productBrand = ref('');

const onReadClick = async () => {
  const input = fileInput.value;
  if (!input || !input.files || !input.files[0]) return;

  const file = input.files[0];
  const imageUrl = URL.createObjectURL(file);

  // Load image into an Image object
  const img = new Image();
  img.src = imageUrl;
  await new Promise((resolve) => { img.onload = resolve; });

  // Draw image to canvas
  const canvas = document.createElement('canvas');
  canvas.width = img.width;
  canvas.height = img.height;
  const ctx = canvas.getContext('2d');
  ctx.drawImage(img, 0, 0);

  // Convert to grayscale
  const imageData = ctx.getImageData(0, 0, canvas.width, canvas.height);
  const data = imageData.data;
  for (let i = 0; i < data.length; i += 4) {
    const avg = 0.299 * data[i] + 0.587 * data[i+1] + 0.114 * data[i+2];
    data[i] = data[i+1] = data[i+2] = avg;
  }
  ctx.putImageData(imageData, 0, 0);

  // Use the canvas as the source for Quagga
  const grayscaleUrl = canvas.toDataURL();

  Quagga.decodeSingle({
    src: grayscaleUrl,
    numOfWorkers: 0,
    inputStream: {
      size: 800
    },
    decoder: {
      readers: [
        "ean_reader",
        "ean_8_reader",
        "code_128_reader",
        "upc_reader",
        "upc_e_reader"
      ]
    }
  }, async function(result) {
    const output = document.getElementById('result');
    if (result && result.codeResult) {
      const barcode = result.codeResult.code;
      output.innerText = `📦 Barcode: ${barcode}`;

      // Fetch OpenFoodFacts
      try {
        const res = await fetch(`https://world.openfoodfacts.org/api/v0/product/${barcode}.json`);
        const data = await res.json();
        const p = data.product;
        if (p) {
          productName.value = p.product_name || 'Unknown';
          productBrand.value = p.brands || 'N/A';
          output.innerHTML += `
            <h2>${productName.value}</h2>
            <p>Brand: ${productBrand.value}</p>
            <p>Ingredients: ${p.ingredients_text || 'N/A'}</p>
            <p>Nutrition per 100g:
              Energy: ${p.nutriments?.energy || 'N/A'} kcal,
              Fat: ${p.nutriments?.fat || 'N/A'}g,
              Carbs: ${p.nutriments?.carbohydrates || 'N/A'}g,
              Protein: ${p.nutriments?.proteins || 'N/A'}g
            </p>
          `;
        } else {
          output.innerText += '\n❌ Product not found!';
          productName.value = '';
          productBrand.value = '';
        }
      } catch (err) {
        output.innerText += `\n⚠️ Error fetching product: ${err.message}`;
      }

    } else {
      output.innerText = '❌ No barcode detected.';
    }
  });
};

const copyToClipboard = (text) => {
  navigator.clipboard.writeText(text).then(() => {
    alert(`Copied to clipboard! ${text}`);
  }).catch(err => {
    console.error('Failed to copy: ', err);
  });
};
</script>

<style scoped>
.result {
  margin-top: 20px;
  padding: 10px;
  border: 1px solid #ccc;
  background-color: #f9f9f9;
  color: #333;
}
</style>