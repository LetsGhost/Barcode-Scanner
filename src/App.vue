<template>
<div>
  <h1>Upload a image of an barcode and get the Item!</h1>
  <input type="file" name="Image" id="upload" accept="image/*" ref="fileInput" />
  <button id="button" @click="onReadClick">READ IT!</button>
  <div id="result" class="result"></div>
</div>
</template>

<script setup>
import { ref } from 'vue';
import Quagga from '@ericblade/quagga2';

const fileInput = ref(null);

const onReadClick = async () => {
  const input = fileInput.value;
  if (!input || !input.files || !input.files[0]) return;

  const file = input.files[0];
  const imageUrl = URL.createObjectURL(file);

  Quagga.decodeSingle({
    src: imageUrl,
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
          output.innerHTML += `
            <h2>${p.product_name || 'Unknown'}</h2>
            <p>Brand: ${p.brands || 'N/A'}</p>
            <p>Ingredients: ${p.ingredients_text || 'N/A'}</p>
            <p>Nutrition per 100g:
              Energy: ${p.nutriments?.energy || 'N/A'} kcal,
              Fat: ${p.nutriments?.fat || 'N/A'}g,
              Carbs: ${p.nutriments?.carbohydrates || 'N/A'}g,
              Protein: ${p.nutriments?.proteins || 'N/A'}g
            </p>
          `;
        } else {
          output.innerText += `\n❌ Product not found.`;
        }
      } catch (err) {
        output.innerText += `\n⚠️ Error fetching product: ${err.message}`;
      }

    } else {
      output.innerText = '❌ No barcode detected.';
    }
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