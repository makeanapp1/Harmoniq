# Harmoniq
import os
import stripe
import librosa
import numpy as np
import soundfile as sf
from flask import Flask, request, jsonify, send_from_directory
from flask_cors import CORS

# Initialize Flask app
app = Flask(__name__)
CORS(app)  # Enable cross-origin requests

# Stripe API Key (Replace with your own)
stripe.api_key = "sk_test_YourStripeSecretKey"

# Configure upload folder
UPLOAD_FOLDER = "uploads"
os.makedirs(UPLOAD_FOLDER, exist_ok=True)
app.config["UPLOAD_FOLDER"] = UPLOAD_FOLDER

# Set pricing for analysis
ANALYSIS_PRICE = 2.99  # Price per song in USD


def analyze_audio(file_path):
    """Extracts key audio features and provides feedback."""
    try:
        y, sr = librosa.load(file_path, sr=None)
        tempo, _ = librosa.beat.beat_track(y=y, sr=sr)
        spectral_centroid = np.mean(librosa.feature.spectral_centroid(y=y, sr=sr))
        rms_energy = np.mean(librosa.feature.rms(y=y))

        feedback = []
        if tempo < 80:
            feedback.append("Consider increasing the tempo for a more energetic feel.")
        elif tempo > 150:
            feedback.append("The tempo is quite fast. Slowing it down might add clarity.")

        if spectral_centroid < 2000:
            feedback.append("Try adding more high-frequency elements to brighten the mix.")
        elif spectral_centroid > 5000:
            feedback.append("The track might be too bright. Balancing with mid-lows could help.")

        return {"tempo": tempo, "rms_energy": rms_energy, "feedback": feedback}
    except Exception as e:
        return {"error": str(e)}


@app.route("/pay", methods=["POST"])
def create_payment():
    """Handles payment via Stripe."""
    try:
        data = request.json
        intent = stripe.PaymentIntent.create(
            amount=int(ANALYSIS_PRICE * 100),  # Convert to cents
            currency="usd",
            automatic_payment_methods={"enabled": True},
        )
        return jsonify({"clientSecret": intent["client_secret"]})
    except Exception as e:
        return jsonify({"error": str(e)}), 400


@app.route("/analyze", methods=["POST"])
def analyze():
    """Handles file upload and analysis after payment."""
    if "file" not in request.files:
        return jsonify({"error": "No file uploaded"}), 400

    file = request.files["file"]
    if not file.filename.endswith(".mp3"):
        return jsonify({"error": "Only MP3 files are allowed"}), 400

    file_path = os.path.join(app.config["UPLOAD_FOLDER"], file.filename)
    file.save(file_path)

    try:
        analysis = analyze_audio(file_path)
        os.remove(file_path)  # Cleanup after processing
        return jsonify(analysis)
    except Exception as e:
        return jsonify({"error": str(e)}), 500


@app.route("/")
def home():
    """Home route for testing."""
    return "HarmoniQ Song & Beat Analyzer API is running!"


if __name__ == "__main__":
    app.run(debug=True)
