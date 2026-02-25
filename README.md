<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Image Generator</title>
    <!-- Imports Tailwind CSS for styling -->
    <script src="https://cdn.tailwindcss.com"></script>
    <!-- Imports the Lucide icon library -->
    <!-- Replaced deprecated 'lucide-icons' with the correct 'lucide' UMD script -->
    <script src="https://unpkg.com/lucide@latest/dist/umd/lucide.js"></script>
    <style>
        /* Custom styles */
        body {
            font-family: 'Inter', sans-serif;
        }
        /* Simple spinner animation */
        .spinner {
            border: 4px solid rgba(0, 0, 0, 0.1);
            width: 48px;
            height: 48px;
            border-radius: 50%;
            border-left-color: #4f46e5; /* Indigo */
            animation: spin 1s ease infinite;
        }
        @keyframes spin {
            0% {
                transform: rotate(0deg);
            }
            100% {
                transform: rotate(360deg);
            }
        }
    </style>
</head>
<body class="bg-gray-100 min-h-screen flex items-center justify-center p-4">
    <div class="bg-white w-full max-w-lg p-6 sm:p-8 rounded-2xl shadow-xl" id="app">
        
        <h1 class="text-3xl font-bold text-center text-gray-800 mb-6">Image Generator</h1>

        <!-- Prompt Input Area -->
        <div class="mb-4">
            <label for="prompt-input" class="block text-sm font-medium text-gray-700 mb-2">Image Prompt:</label>
            <textarea id="prompt-input" rows="3" class="w-full p-3 border border-gray-300 rounded-lg focus:ring-2 focus:ring-indigo-500 focus:border-indigo-500 transition-shadow" placeholder="e.g., A red panda wearing a small astronaut helmet...">A sleek, modern logo for a 'Key System', futuristic, digital, on a dark background.</textarea>
        </div>

        <!-- Generate Button -->
        <button id="generate-button" class="w-full bg-indigo-600 text-white font-medium py-3 px-6 rounded-lg shadow-md hover:bg-indigo-700 transition-all flex items-center justify-center disabled:opacity-50">
            <i data-lucide="image" class="inline-block h-5 w-5 mr-2 -mt-0.5"></i>
            Generate Image
        </button>

        <!-- Loading and Image Area -->
        <div id="output-area" class="mt-6 w-full aspect-square bg-gray-50 rounded-lg border border-gray-200 flex items-center justify-center overflow-hidden">
            <!-- Loader -->
            <div id="loader" class="hidden flex-col items-center text-gray-500">
                <div class="spinner"></div>
                <p class="mt-4 text-sm font-medium">Generating your image...</p>
            </div>
            
            <!-- Image Display -->
            <img id="result-image" src="" alt="Generated image" class="hidden w-full h-full object-cover">

            <!-- Initial Message -->
            <div id="initial-message" class="text-center text-gray-400 p-4">
                <i data-lucide="image-play" class="h-16 w-16 mx-auto mb-3"></i>
                <p>Your generated image will appear here.</p>
            </div>

            <!-- Error Message -->
            <div id="error-message" class="hidden text-center text-red-500 p-4">
                <i data-lucide="alert-triangle" class="h-12 w-12 mx-auto mb-3"></i>
                <p class="font-medium">Image generation failed.</p>
                <p class="text-sm" id="error-details"></p>
            </div>
        </div>
    </div>

    <script>
        document.addEventListener('DOMContentLoaded', () => {
            // --- DOM Elements ---
            const promptInput = document.getElementById('prompt-input');
            const generateButton = document.getElementById('generate-button');
            const outputArea = document.getElementById('output-area');
            const loader = document.getElementById('loader');
            const resultImage = document.getElementById('result-image');
            const initialMessage = document.getElementById('initial-message');
            const errorMessage = document.getElementById('error-message');
            const errorDetails = document.getElementById('error-details');

            const apiKey = ""; // API key is handled by the environment
            const apiUrl = `https://generativelanguage.googleapis.com/v1beta/models/imagen-3.0-generate-002:predict?key=${apiKey}`;

            /**
             * Shows the correct state in the output area.
             * @param {string} state - 'initial', 'loading', 'image', or 'error'.
             */
            function showState(state) {
                initialMessage.classList.add('hidden');
                loader.classList.add('hidden');
                loader.classList.remove('flex');
                resultImage.classList.add('hidden');
                errorMessage.classList.add('hidden');

                if (state === 'loading') {
                    loader.classList.remove('hidden');
                    loader.classList.add('flex');
                } else if (state === 'image') {
                    resultImage.classList.remove('hidden');
                } else if (state === 'error') {
                    errorMessage.classList.remove('hidden');
                } else { // initial
                    initialMessage.classList.remove('hidden');
                }
            }

            /**
             * Fetches the image from the API with exponential backoff.
             */
            async function generateImage() {
                const prompt = promptInput.value;
                if (!prompt) {
                    alert('Please enter a prompt.');
                    return;
                }

                generateButton.disabled = true;
                generateButton.textContent = 'Generating...';
                showState('loading');
                
                const payload = {
                    instances: [{ prompt: prompt }],
                    parameters: { "sampleCount": 1 }
                };

                let attempt = 0;
                const maxAttempts = 5;
                let delay = 1000; // 1 second

                while (attempt < maxAttempts) {
                    try {
                        const response = await fetch(apiUrl, {
                            method: 'POST',
                            headers: { 'Content-Type': 'application/json' },
                            body: JSON.stringify(payload)
                        });

                        if (!response.ok) {
                            // Check for rate limiting or server errors
                            if (response.status === 429 || response.status >= 500) {
                                // This is a retriable error
                                throw new Error(`Retriable error: ${response.statusText}`);
                            } else {
                                // This is a permanent error
                                const errorData = await response.json();
                                throw new Error(`API Error: ${errorData.error?.message || response.statusText}`);
                            }
                        }

                        const result = await response.json();

                        if (result.predictions && result.predictions.length > 0 && result.predictions[0].bytesBase64Encoded) {
                            const base64Data = result.predictions[0].bytesBase64Encoded;
                            resultImage.src = `data:image/png;base64,${base64Data}`;
                            showState('image');
                            resetButton();
                            return; // Success, exit the loop
                        } else {
                            throw new Error('Invalid response structure from API.');
                        }

                    } catch (error) {
                        console.error(`Attempt ${attempt + 1} failed:`, error.message);
                        attempt++;
                        if (attempt >= maxAttempts || !error.message.startsWith('Retriable error')) {
                            // Non-retriable error or max attempts reached
                            showState('error');
                            errorDetails.textContent = error.message;
                            resetButton();
                            return; // Exit loop
                        }
                        // Wait before retrying
                        await new Promise(resolve => setTimeout(resolve, delay));
                        delay *= 2; // Exponential backoff
                    }
                }
            }
            
            function resetButton() {
                generateButton.disabled = false;
                generateButton.innerHTML = `
                    <i data-lucide="image" class="inline-block h-5 w-5 mr-2 -mt-0.5"></i>
                    Generate Image
                `;
                lucide.createIcons(); // Re-render icon
            }

            // --- Attach Event Listeners ---
            generateButton.addEventListener('click', generateImage);

            // --- Initial Icon Render ---
            lucide.createIcons();
        });
    </script>
</body>
</html>


