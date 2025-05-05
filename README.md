# tarekimport React, { useState } from 'react';
import { Button, Image, View, Text, ActivityIndicator, StyleSheet } from 'react-native';
import * as ImagePicker from 'expo-image-picker';
import axios from 'axios';

// Put your real OpenAI API key here
const OPENAI_API_KEY = 'YOUR_REAL_OPENAI_API_KEY';

function extractCoordinates(aiText: string): { x: number, y: number } | null {
  const match = aiText.match(/(\d+)[,\s]+(\d+)/);
  if (match) {
    return { x: parseInt(match[1]), y: parseInt(match[2]) };
  }
  return null;
}

const analyzeScreen = async (screenshotUri: string) => {
  const response = await axios.post(
    'https://api.openai.com/v1/chat/completions',
    {
      model: "gpt-4-vision-preview",
      messages: [
        {
          role: "user",
          content: [
            { type: "text", text: "Where to click? Give coordinates only." },
            { type: "image_url", image_url: { url: screenshotUri } }
          ]
        }
      ],
      max_tokens: 300
    },
    {
      headers: {
        'Authorization': `Bearer ${OPENAI_API_KEY}`,
        'Content-Type': 'application/json'
      }
    }
  );
  return response.data;
};

export default function App() {
  const [image, setImage] = useState<string | null>(null);
  const [result, setResult] = useState<any>(null);
  const [coords, setCoords] = useState<{ x: number, y: number } | null>(null);
  const [loading, setLoading] = useState(false);

  const pickImage = async () => {
    const res = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsEditing: false,
      quality: 1,
    });
    if (!res.canceled) {
      setImage(res.assets[0].uri);
      setResult(null);
      setCoords(null);
    }
  };

  const handleAnalyze = async () => {
    if (!image) return;
    setLoading(true);
    try {
      const aiResult = await analyzeScreen(image);
      setResult(aiResult);
      const text = aiResult.choices?.[0]?.message?.content || '';
      const c = extractCoordinates(text);
      setCoords(c);
    } catch (e: any) {
      setResult({ error: String(e) });
    }
    setLoading(false);
  };

  // For overlay, assume image is displayed at 200x400
  const imgWidth = 200, imgHeight = 400;

  return (
    <View style={styles.container}>
      <Button title="Pick Screenshot" onPress={pickImage} />
      {image && (
        <View style={{ width: imgWidth, height: imgHeight }}>
          <Image source={{ uri: image }} style={styles.image} />
          {coords && (
            <View
              style={{
                position: 'absolute',
                left: coords.x - 10,
                top: coords.y - 10,
                width: 20,
                height: 20,
                borderRadius: 10,
                backgroundColor: 'red',
                opacity: 0.7,
                borderWidth: 2,
                borderColor: 'white',
              }}
            />
          )}
        </View>
      )}
      {image && <Button title="Ask AI: Where to click?" onPress={handleAnalyze} />}
      {loading && <ActivityIndicator size="large" />}
      {result && <Text style={styles.result}>{JSON.stringify(result, null, 2)}</Text>}
    </View>
  );
}

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: 'center', justifyContent: 'center', padding: 20 },
  image: { width: 200, height: 400, margin: 10, resizeMode: 'contain' },
  result: { marginTop: 20, color: 'black', fontSize: 12 }
});
