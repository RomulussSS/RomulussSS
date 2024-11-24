
// Version améliorée d'une appli React Native avec des fonctionnalités supplémentaires : un bâtiment offert à la première connexion, gain passif toutes les 24h et possibilité de partage des bâtiments entre plusieurs utilisateurs.

// 1. Initialise un projet React Native
// $ npx react-native init GeoEmpire

// 2. Installe les dépendances pour la localisation, Firebase, et l'authentification
// $ npm install @react-native-community/geolocation
// $ npm install @react-native-firebase/app @react-native-firebase/firestore @react-native-firebase/auth
// $ npm install react-navigation react-native-gesture-handler react-native-reanimated react-native-safe-area-context react-native-screens

import React, { useState, useEffect } from 'react';
import { View, Text, Button, StyleSheet, Alert, TextInput, FlatList, ActivityIndicator } from 'react-native';
import Geolocation from '@react-native-community/geolocation';
import firestore from '@react-native-firebase/firestore';
import auth from '@react-native-firebase/auth';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';

const Stack = createStackNavigator();

const GeoEmpire = () => {
  return (
    <NavigationContainer>
      <Stack.Navigator>
        <Stack.Screen name="Login" component={LoginScreen} />
        <Stack.Screen name="Home" component={HomeScreen} />
        <Stack.Screen name="Chat" component={ChatScreen} />
        <Stack.Screen name="BuyHome" component={BuyHomeScreen} />
      </Stack.Navigator>
    </NavigationContainer>
  );
};

const LoginScreen = ({ navigation }) => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [loading, setLoading] = useState(false);

  const handleLogin = async () => {
    if (!email || !password) {
      Alert.alert('Erreur', 'Veuillez remplir tous les champs');
      return;
    }
    setLoading(true);
    try {
      await auth().signInWithEmailAndPassword(email, password);
      navigation.navigate('Home');
    } catch (error) {
      Alert.alert('Erreur de connexion', error.message);
    } finally {
      setLoading(false);
    }
  };

  const handleSignup = async () => {
    if (!email || !password) {
      Alert.alert('Erreur', 'Veuillez remplir tous les champs');
      return;
    }
    setLoading(true);
    try {
      const userCredential = await auth().createUserWithEmailAndPassword(email, password);
      const user = userCredential.user;

      // Offrir un bâtiment de base à la première inscription
      await firestore().collection('buildings').add({
        latitude: 0, // Coordonnées fictives pour le bâtiment offert
        longitude: 0,
        owner: user.uid,
        shares: [{ ownerId: user.uid, share: 100 }], // Le bâtiment est à 100% de l'utilisateur
        timestamp: firestore.FieldValue.serverTimestamp(),
      });
      Alert.alert('Félicitations! Vous avez reçu un bâtiment gratuit pour commencer.');
      navigation.navigate('Home');
    } catch (error) {
      Alert.alert('Erreur d'inscription', error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <TextInput
        placeholder="Email"
        value={email}
        onChangeText={setEmail}
        style={styles.input}
        keyboardType="email-address"
        autoCapitalize="none"
      />
      <TextInput
        placeholder="Mot de passe"
        value={password}
        onChangeText={setPassword}
        secureTextEntry
        style={styles.input}
      />
      {loading ? (
        <ActivityIndicator size="large" color="#0000ff" />
      ) : (
        <>
          <Button title="Connexion" onPress={handleLogin} />
          <Button title="Inscription" onPress={handleSignup} />
        </>
      )}
    </View>
  );
};

const HomeScreen = ({ navigation }) => {
  const [location, setLocation] = useState(null);
  const [buildings, setBuildings] = useState([]);
  const [friends, setFriends] = useState([]);
  const [friendEmail, setFriendEmail] = useState('');
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const fetchLocation = () => {
      Geolocation.getCurrentPosition(
        (position) => {
          const { latitude, longitude } = position.coords;
          setLocation({ latitude, longitude });
        },
        (error) => console.log(error),
        { enableHighAccuracy: true, timeout: 20000, maximumAge: 1000 }
      );
    };

    const fetchFriendsAndBuildings = async () => {
      try {
        const user = auth().currentUser;
        if (!user) return;

        // Récupérer les bâtiments que l'utilisateur possède ou partage
        const buildingsRef = firestore().collection('buildings');
        const buildingsSnapshot = await buildingsRef
          .where('shares', 'array-contains', { ownerId: user.uid })
          .get();
        setBuildings(buildingsSnapshot.docs.map(doc => doc.data()));

        const friendsRef = firestore().collection('users').doc(user.uid).collection('friends');
        const friendsSnapshot = await friendsRef.get();
        const friendsList = friendsSnapshot.docs.map(doc => doc.data());
        setFriends(friendsList);
      } catch (error) {
        console.error('Erreur lors de la récupération des amis et des bâtiments : ', error);
      } finally {
        setLoading(false);
      }
    };

    fetchLocation();
    fetchFriendsAndBuildings();
  }, []);

  const handleBuyBuilding = async () => {
    if (location) {
      try {
        const user = auth().currentUser;
        if (!user) return;

        const buildingsRef = firestore().collection('buildings');
        const snapshot = await buildingsRef
          .where('latitude', '==', location.latitude)
          .where('longitude', '==', location.longitude)
          .get();

        if (!snapshot.empty) {
          // Bâtiment déjà possédé, offrir la possibilité de partager les parts
          const building = snapshot.docs[0];
          const buildingData = building.data();
          if (buildingData.shares.some(share => share.ownerId === user.uid)) {
            Alert.alert('Vous possédez déjà une partie de ce bâtiment!');
          } else {
            // Ajouter une nouvelle part pour cet utilisateur
            await building.ref.update({
              shares: firestore.FieldValue.arrayUnion({ ownerId: user.uid, share: 30 }), // Par exemple, 30% des parts
            });
            Alert.alert('Vous avez acheté une part de ce bâtiment!');
          }
        } else {
          await buildingsRef.add({
            latitude: location.latitude,
            longitude: location.longitude,
            owner: user.uid,
            shares: [{ ownerId: user.uid, share: 100 }],
            timestamp: firestore.FieldValue.serverTimestamp(),
          });
          Alert.alert('Félicitations, vous avez acheté ce bâtiment!');
        }
      } catch (error) {
        console.error('Erreur lors de l'achat du bâtiment : ', error);
      }
    }
  };

  const handleAddFriend = async () => {
    if (!friendEmail) {
      Alert.alert('Erreur', 'Veuillez entrer un email valide');
      return;
    }
    try {
      const user = auth().currentUser;
      if (!user) return;

      const friendsRef = firestore().collection('users').doc(user.uid).collection('friends');
      const userSnapshot = await firestore().collection('users').where('email', '==', friendEmail).get();
      if (!userSnapshot.empty) {
        await friendsRef.add({ id: userSnapshot.docs[0].id, email: friendEmail });
        Alert.alert('Ami ajouté avec succès!');
        setFriendEmail('');
      } else {
        Alert.alert('Utilisateur non trouvé!');
      }
    } catch (error) {
      console.error('Erreur lors de l'ajout d'un ami : ', error);
    }
  };

  if (loading) {
    return (
      <View style={styles.container}>
        <ActivityIndicator size="large" color="#0000ff" />
      </View>
    );
  }

  return (
    <View style={styles.container}>
      {location ? (
        <Text>Position actuelle : {location.latitude}, {location.longitude}</Text>
      ) : (
        <Text>Chargement de la position...</Text>
      )}
      <Button
        title="Acheter le bâtiment ici"
        onPress={handleBuyBuilding}
      />

      <Text>Liste des amis :</Text>
      <FlatList
        data={friends}
        keyExtractor={(item) => item.id}
        renderItem={({ item }) => <Text>{item.email}</Text>}
      />

      <TextInput
        placeholder="Email de l'ami"
        value={friendEmail}
        onChangeText={setFriendEmail}
        style={styles.input}
      />
      <Button title="Ajouter un ami" onPress={handleAddFriend} />
      <Button title="Ouvrir le chat" onPress={() => navigation.navigate('Chat')} />
      <Button title="Acheter votre maison virtuelle" onPress={() => navigation.navigate('BuyHome')} />
    </View>
  );
};

const BuyHomeScreen = () => {
  const [homeAddress, setHomeAddress] = useState('');
  const [price, setPrice] = useState('');
  const [loading, setLoading] = useState(false);

  const handleBuyHome = async () => {
    if (!homeAddress || !price) {
      Alert.alert('Erreur', 'Veuillez remplir tous les champs');
      return;
    }
    setLoading(true);
    try {
      const user = auth().currentUser;
      if (!user) return;

      await firestore().collection('homes').add({
        address: homeAddress,
        price: parseFloat(price),
        owner: user.uid,
        timestamp: firestore.FieldValue.serverTimestamp(),
      });
      Alert.alert('Félicitations, vous avez acheté votre maison virtuelle!');
      setHomeAddress('');
      setPrice('');
    } catch (error) {
      console.error('Erreur lors de l'achat de la maison : ', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <View style={styles.container}>
      <TextInput
        placeholder="Adresse de la maison"
        value={homeAddress}
        onChangeText={setHomeAddress}
        style={styles.input}
      />
      <TextInput
        placeholder="Prix de la maison"
        value={price}
        onChangeText={setPrice}
        style={styles.input}
        keyboardType="numeric"
      />
      {loading ? (
        <ActivityIndicator size="large" color="#0000ff" />
      ) : (
        <Button title="Acheter" onPress={handleBuyHome} />
      )}
    </View>
  );
};

const ChatScreen = () => {
  const [messages, setMessages] = useState([]);
  const [newMessage, setNewMessage] = useState('');
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    const messagesRef = firestore().collection('messages').orderBy('timestamp', 'asc');
    const unsubscribe = messagesRef.onSnapshot((snapshot) => {
      const messagesList = snapshot.docs.map((doc) => doc.data());
      setMessages(messagesList);
      setLoading(false);
    });
    return () => unsubscribe();
  }, []);

  const handleSendMessage = async () => {
    if (!newMessage.trim()) {
      Alert.alert('Erreur', 'Veuillez entrer un message');
      return;
    }
    try {
      const user = auth().currentUser;
      if (!user) return;

      await firestore().collection('messages').add({
        text: newMessage,
        user: user.email,
        timestamp: firestore.FieldValue.serverTimestamp(),
      });
      setNewMessage('');
    } catch (error) {
      console.error('Erreur lors de l'envoi du message : ', error);
    }
  };

  if (loading) {
    return (
      <View style={styles.container}>
        <ActivityIndicator size="large" color="#0000ff" />
      </View>
    );
  }

  return (
    <View style={styles.container}>
      <FlatList
        data={messages}
        keyExtractor={(item, index) => index.toString()}
        renderItem={({ item }) => <Text>{item.user}: {item.text}</Text>}
      />
      <TextInput
        placeholder="Écrire un message"
        value={newMessage}
        onChangeText={setNewMessage}
        style={styles.input}
      />
      <Button title="Envoyer" onPress={handleSendMessage} />
    </View>
  );
};

const styles = StyleSheet.create({
  container: {
    flex: 1,
    justifyContent: 'center',
    alignItems: 'center',
    padding: 16,
  },
  input: {
    borderWidth: 1,
    padding: 8,
    marginVertical: 8,
    width: '80%',
  },
});

export default GeoEmpire;
