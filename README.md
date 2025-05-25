Compris ! Tu souhaites donc que je reprenne l'exemple précédent, mais en me concentrant **uniquement sur la partie backend Node.js** pour l'intégration avec FedaPay, sans la partie frontend React en un seul fichier.

Cela signifie que nous allons créer un serveur Node.js avec des endpoints API que tu pourras ensuite appeler depuis n'importe quel client (Postman, une application frontend séparée, etc.) pour tester la création de transactions FedaPay.

**Objectif :** Un serveur Node.js qui expose un endpoint pour créer une transaction FedaPay. Les informations (clés API, détails de la transaction) seront envoyées dans le corps de la requête.

Voici le code pour `fedapay-server-test.js` :

```javascript
// fedapay-server-test.js

const express = require('express');
const bodyParser = require('body-parser');
const FedaPay = require('fedapay'); // SDK officiel FedaPay
const dotenv = require('dotenv');

dotenv.config(); // Charge les variables de .env (recommandé pour les clés en production)

const app = express();
const PORT = process.env.PORT || 3001; // Port différent si vous exécutez aussi l'autre app

app.use(bodyParser.json()); // Accepter les requêtes JSON

// Endpoint pour créer une transaction FedaPay
app.post('/api/fedapay/create-transaction', async (req, res) => {
    const {
        apiKeySecret, // Clé secrète, fournie dans la requête pour ce test
        environment,  // 'sandbox' ou 'live', fourni dans la requête
        amount,
        description,
        currencyIso,  // ex: 'XOF'
        customer // Objet client: { firstname, lastname, email, phone_number: { number, country } }
    } = req.body;

    // Validation basique des entrées
    if (!apiKeySecret || !environment || !amount || !description || !currencyIso || !customer) {
        return res.status(400).json({
            success: false,
            message: "Paramètres manquants. Assurez-vous de fournir apiKeySecret, environment, amount, description, currencyIso, et customer."
        });
    }

    try {
        // Configuration dynamique de FedaPay SDK pour ce test.
        // EN PRODUCTION : Ces valeurs viendraient de process.env et seraient configurées au démarrage du serveur.
        // FedaPay.setApiKey(process.env.FEDAPAY_SECRET_KEY_SANDBOX); // Exemple
        // FedaPay.setEnvironment(process.env.FEDAPAY_ENVIRONMENT || 'sandbox'); // Exemple
        FedaPay.setApiKey(apiKeySecret);
        FedaPay.setEnvironment(environment);

        console.log(`[FedaPay Server] Tentative de création de transaction en mode ${environment}`);
        console.log(`[FedaPay Server] Montant: ${amount} ${currencyIso}, Description: ${description}`);
        console.log(`[FedaPay Server] Client: ${customer.firstname} ${customer.lastname}`);

        const transaction = await FedaPay.Transaction.create({
            description: description,
            amount: parseInt(amount), // Assurez-vous que le montant est un entier
            currency: { iso: currencyIso },
            customer: {
                firstname: customer.firstname,
                lastname: customer.lastname,
                email: customer.email,
                phone_number: { // L'objet phone_number est attendu par FedaPay
                    number: customer.phone_number.number,
                    country: customer.phone_number.country // ex: 'BJ'
                }
            },
            // Dans une vraie application, vous définiriez un callback_url pour être notifié par FedaPay
            // callback_url: 'https://votreserveur.com/fedapay/webhook-callback',
        });

        // Une fois la transaction créée, vous devez générer un token pour le paiement.
        // Ce token est ensuite utilisé par FedaPay Checkout (côté client) ou pour une redirection.
        const tokenObject = await transaction.generateToken();

        console.log(`[FedaPay Server] Transaction FedaPay créée avec ID: ${transaction.id}`);
        console.log(`[FedaPay Server] Token de paiement FedaPay: ${tokenObject.token}`);
        // console.log(`[FedaPay Server] URL de paiement (pour redirection): ${tokenObject.url}`);

        res.status(200).json({
            success: true,
            message: 'Transaction FedaPay initiée avec succès.',
            transactionId: transaction.id,
            paymentToken: tokenObject.token,
            paymentUrl: tokenObject.url // L'URL que le client peut utiliser pour payer
        });

    } catch (error) {
        console.error('[FedaPay Server] Erreur FedaPay:', error.message);
        if (error.response && error.response.data) {
            console.error('[FedaPay Server] Détails de l'erreur FedaPay:', error.response.data);
        }
        res.status(500).json({
            success: false,
            message: 'Erreur lors de la création de la transaction FedaPay.',
            error: error.message,
            details: error.response ? error.response.data : null
        });
    }
});

// Endpoint pour simuler la réception d'un webhook/callback de FedaPay (très simplifié)
// EN PRODUCTION: Cet endpoint doit être sécurisé et vérifier l'authenticité de la requête FedaPay.
app.post('/api/fedapay/webhook', (req, res) => {
    const event = req.body; // FedaPay envoie un objet JSON
    console.log('[FedaPay Server] Webhook FedaPay reçu:', JSON.stringify(event, null, 2));

    // Exemple de traitement d'événement
    if (event.name === 'transaction.approved') {
        const transactionData = event.data;
        console.log(`[FedaPay Server] Transaction ID ${transactionData.id} approuvée !`);
        // Ici, vous mettriez à jour le statut de la commande dans votre base de données,
        // envoyeriez des emails de confirmation, etc.
    } else if (event.name === 'transaction.canceled' || event.name === 'transaction.failed') {
        const transactionData = event.data;
        console.log(`[FedaPay Server] Transaction ID ${transactionData.id} échouée ou annulée.`);
        // Gérer l'échec
    }

    // Répondre à FedaPay pour accuser réception du webhook
    res.status(200).send('Webhook reçu');
});


app.listen(PORT, () => {
    console.log(`Serveur Node.js pour test FedaPay démarré sur http://localhost:${PORT}`);
    console.log(`Utilisez un client API (Postman) pour envoyer une requête POST à /api/fedapay/create-transaction`);
});
```

**Comment utiliser ce serveur Node.js :**

1.  **Sauvegarder :** Enregistrez le code ci-dessus dans un fichier nommé `fedapay-server-test.js`.
2.  **Installer les dépendances :**
    Si vous ne l'avez pas déjà fait dans le même répertoire :
    ```bash
    npm init -y
    npm install express body-parser fedapay dotenv
    ```
3.  **Obtenir vos clés API FedaPay Sandbox :**
    Comme précédemment, assurez-vous d'avoir vos clés "Publique" et "Secrète" de l'environnement Sandbox de FedaPay. Pour ce script serveur, c'est la **clé secrète** qui sera principalement utilisée.

4.  **Exécuter le serveur :**
    ```bash
    node fedapay-server-test.js
    ```
    Le serveur démarrera et écoutera sur le port 3001 (ou celui que vous avez défini).

5.  **Tester avec un client API (comme Postman) :**
    *   Ouvrez Postman (ou un outil similaire).
    *   Créez une nouvelle requête **POST**.
    *   **URL de la requête :** `http://localhost:3001/api/fedapay/create-transaction`
    *   **Headers :** Ajoutez un header `Content-Type` avec la valeur `application/json`.
    *   **Body :** Sélectionnez l'option "raw" et choisissez "JSON". Collez un corps de requête comme celui-ci, en remplaçant les valeurs par vos informations de test :

        ```json
        {
            "apiKeySecret": "sk_sandbox_VOTRE_CLE_SECRETE_FEDAPAY_ICI",
            "environment": "sandbox",
            "amount": 150,
            "description": "Achat de test depuis serveur Node.js",
            "currencyIso": "XOF",
            "customer": {
                "firstname": "Test",
                "lastname": "Utilisateur",
                "email": "test.utilisateur@example.com",
                "phone_number": {
                    "number": "91234567",
                    "country": "BJ"
                }
            }
        }
        ```
        **N'oubliez pas de remplacer `"sk_sandbox_VOTRE_CLE_SECRETE_FEDAPAY_ICI"` par votre vraie clé secrète sandbox.**

    *   **Envoyer la requête.**

6.  **Résultat attendu :**
    *   Le serveur Node.js devrait recevoir la requête, configurer le SDK FedaPay, créer une transaction, et générer un token.
    *   Vous devriez recevoir une réponse JSON de votre serveur similaire à ceci :
        ```json
        {
            "success": true,
            "message": "Transaction FedaPay initiée avec succès.",
            "transactionId": "TRN_XXXXXXXXXXXXX", // L'ID de la transaction
            "paymentToken": "PKT_XXXXXXXXXXXXX", // Le token de paiement
            "paymentUrl": "https://checkout-sandbox.fedapay.com/pay/PKT_XXXXXXXXXXXXX" // L'URL pour payer
        }
        ```
    *   Vous pouvez copier `paymentUrl` et l'ouvrir dans un navigateur pour procéder au paiement simulé avec les identifiants de test FedaPay.
    *   Vérifiez les logs dans votre console Node.js pour voir les étapes de traitement.

**Endpoint Webhook ( `/api/fedapay/webhook` ) :**

*   Ce serveur inclut aussi un endpoint basique pour recevoir des webhooks de FedaPay.
*   Dans votre tableau de bord FedaPay (Sandbox), vous pouvez configurer une URL de webhook pour pointer vers `http://localhost:3001/api/fedapay/webhook` (vous aurez besoin d'un outil comme ngrok pour exposer votre localhost à internet si FedaPay doit vous atteindre depuis leurs serveurs).
*   Quand une transaction change de statut (approuvée, annulée, etc.), FedaPay enverra une notification à cet endpoint. C'est crucial en production pour une confirmation fiable des paiements.

Ce serveur Node.js est une base plus propre pour interagir avec FedaPay côté backend. Vous pouvez maintenant construire une application frontend séparée (React, Vue, Angular, HTML simple) qui appelle cet endpoint `/api/fedapay/create-transaction` pour initier les paiements.
