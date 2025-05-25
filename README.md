Okay, créer une mini-application Node.js avec un frontend React (très basique, servi par Node) en un seul fichier pour tester l'API FedaPay est un bon exercice. L'objectif est de simplifier au maximum pour se concentrer sur l'intégration.

**Important :**

1.  **Sécurité des Clés :** Dans une application réelle, votre clé API secrète ne devrait **JAMAIS** être exposée côté client (React). Elle doit rester sur le serveur (Node.js). Pour ce test, nous allons la saisir dans le formulaire, mais le traitement se fera côté serveur.
2.  **Environnement de Test :** Utilisez impérativement vos clés API **sandbox (test)** de FedaPay pour cet exercice. N'utilisez jamais vos clés live pour des tests.
3.  **"Envoyer de l'argent" :** L'API FedaPay est principalement conçue pour que *vous* (le marchand) puissiez *recevoir* des paiements de clients. "Envoyer de l'argent" directement à un autre compte FedaPay comme un transfert P2P via l'API de transaction standard est moins direct. L'API de "Payout" (versement) serait plus appropriée pour envoyer de l'argent depuis votre compte FedaPay vers des comptes bancaires ou mobiles, mais elle a ses propres prérequis.
    Pour ce test, nous allons simuler un **paiement client** vers le compte FedaPay associé à vos clés API. L'argent "reçu" sera donc sur *votre* compte FedaPay de test.
4.  **Single File avec React :** Pour garder cela en un seul fichier, nous allons servir une page HTML simple depuis Node.js, et cette page HTML inclura React et ReactDOM via CDN, ainsi que notre code React directement dans une balise `<script type="text/babel">`. Babel standalone sera aussi inclus via CDN pour transpiler le JSX à la volée (ce n'est pas pour la production).

Voici le code. Copiez-le dans un fichier nommé `fedapay-test-app.js` :

```javascript
// fedapay-test-app.js

const express = require('express');
const bodyParser = require('body-parser');
const FedaPay = require('fedapay'); // SDK officiel FedaPay
const path = require('path');
const dotenv = require('dotenv'); // Pour gérer les variables d'environnement

dotenv.config(); // Charge les variables de .env si présentes (non utilisé directement ici mais bonne pratique)

const app = express();
const PORT = process.env.PORT || 3000;

app.use(bodyParser.urlencoded({ extended: true }));
app.use(bodyParser.json());

// Servir la page HTML principale
app.get('/', (req, res) => {
    res.send(`
        <!DOCTYPE html>
        <html lang="fr">
        <head>
            <meta charset="UTF-8">
            <meta name="viewport" content="width=device-width, initial-scale=1.0">
            <title>Test Intégration FedaPay</title>
            <script src="https://unpkg.com/react@17/umd/react.development.js"></script>
            <script src="https://unpkg.com/react-dom@17/umd/react-dom.development.js"></script>
            <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
            <script src="https://cdn.fedapay.com/checkout.js?v=1.1.7"></script>
            <style>
                body { font-family: sans-serif; margin: 20px; background-color: #f4f4f4; color: #333; }
                .container { background-color: white; padding: 20px; border-radius: 8px; box-shadow: 0 0 10px rgba(0,0,0,0.1); max-width: 500px; margin: auto; }
                h1 { text-align: center; color: #007bff; }
                label { display: block; margin-top: 15px; margin-bottom: 5px; font-weight: bold; }
                input[type="text"], input[type="email"], input[type="number"], select {
                    width: calc(100% - 22px); padding: 10px; margin-bottom: 10px; border: 1px solid #ddd; border-radius: 4px;
                }
                button {
                    background-color: #007bff; color: white; padding: 12px 20px; border: none; border-radius: 4px;
                    cursor: pointer; font-size: 16px; width: 100%; transition: background-color 0.3s;
                }
                button:hover { background-color: #0056b3; }
                .error, .success {
                    padding: 10px; margin-top: 15px; border-radius: 4px; text-align: center;
                }
                .error { background-color: #f8d7da; color: #721c24; border: 1px solid #f5c6cb; }
                .success { background-color: #d4edda; color: #155724; border: 1px solid #c3e6cb; }
            </style>
        </head>
        <body>
            <div id="root"></div>

            <script type="text/babel">
                function App() {
                    const [apiKeyPublic, setApiKeyPublic] = React.useState('');
                    const [apiKeySecret, setApiKeySecret] = React.useState('');
                    const [environment, setEnvironment] = React.useState('sandbox'); // 'sandbox' ou 'live'
                    const [amount, setAmount] = React.useState(100); // Montant en CFA par défaut
                    const [description, setDescription] = React.useState('Test de paiement');
                    const [customerFirstname, setCustomerFirstname] = React.useState('John');
                    const [customerLastname, setCustomerLastname] = React.useState('Doe');
                    const [customerEmail, setCustomerEmail] = React.useState('john.doe@example.com');
                    const [customerPhone, setCustomerPhone] = React.useState('90000000'); // Numéro de téléphone béninois
                    const [currency, setCurrency] = React.useState('XOF'); // CFA

                    const [message, setMessage] = React.useState('');
                    const [messageType, setMessageType] = React.useState(''); // 'error' ou 'success'

                    const handleSubmit = async (event) => {
                        event.preventDefault();
                        setMessage('');
                        setMessageType('');

                        if (!apiKeyPublic || !apiKeySecret) {
                            setMessage('Veuillez entrer vos clés API publiques et secrètes.');
                            setMessageType('error');
                            return;
                        }

                        try {
                            const response = await fetch('/create-transaction', {
                                method: 'POST',
                                headers: { 'Content-Type': 'application/json' },
                                body: JSON.stringify({
                                    apiKeyPublic,
                                    apiKeySecret,
                                    environment,
                                    amount: parseInt(amount),
                                    description,
                                    currency,
                                    customer: {
                                        firstname: customerFirstname,
                                        lastname: customerLastname,
                                        email: customerEmail,
                                        phone_number: {
                                            number: customerPhone,
                                            country: 'BJ' // Bénin par défaut pour FedaPay
                                        }
                                    }
                                })
                            });

                            const data = await response.json();

                            if (response.ok && data.token) {
                                setMessage('Transaction initiée. Redirection vers FedaPay...');
                                setMessageType('success');
                                
                                // Utilisation de FedaPay Checkout (pop-up)
                                FedaPay.checkout({
                                    public_key: apiKeyPublic,
                                    transaction: {
                                        token: data.token,
                                        // Si vous avez un ID de transaction, vous pouvez aussi l'utiliser
                                        // id: data.transactionId 
                                    },
                                    container: '#fedapay-checkout-container', // Optionnel: si vous avez un div dédié
                                    onComplete: function(resp) {
                                        console.log('FedaPay Checkout Response:', resp);
                                        if (resp.reason === FedaPay.CHECKOUT_COMPLETED) {
                                            setMessage(\`Paiement terminé avec succès ! Statut : \${resp.status}\`);
                                            setMessageType('success');
                                            // Idéalement, vérifiez le statut de la transaction sur votre backend ici
                                        } else if (resp.reason === FedaPay.CHECKOUT_DISMISSED) {
                                            setMessage('Le processus de paiement a été annulé par l'utilisateur.');
                                            setMessageType('error');
                                        } else {
                                            setMessage(\`Le paiement a échoué ou a été annulé. Raison : \${resp.reason}, Statut: \${resp.status}\`);
                                            setMessageType('error');
                                        }
                                    }
                                });
                                // Si vous préfériez une redirection: window.location.href = data.checkoutUrl;
                            } else {
                                setMessage(data.message || 'Erreur lors de la création de la transaction.');
                                setMessageType('error');
                            }
                        } catch (error) {
                            console.error('Erreur:', error);
                            setMessage('Une erreur technique est survenue.');
                            setMessageType('error');
                        }
                    };

                    return (
                        <div className="container">
                            <h1>Test Intégration FedaPay</h1>
                            <form onSubmit={handleSubmit}>
                                <label htmlFor="apiKeyPublic">Clé API Publique FedaPay:</label>
                                <input type="text" id="apiKeyPublic" value={apiKeyPublic} onChange={e => setApiKeyPublic(e.target.value)} required />

                                <label htmlFor="apiKeySecret">Clé API Secrète FedaPay:</label>
                                <input type="text" id="apiKeySecret" value={apiKeySecret} onChange={e => setApiKeySecret(e.target.value)} required />
                                
                                <label htmlFor="environment">Environnement FedaPay:</label>
                                <select id="environment" value={environment} onChange={e => setEnvironment(e.target.value)}>
                                    <option value="sandbox">Sandbox (Test)</option>
                                    <option value="live">Live (Production - ATTENTION)</option>
                                </select>

                                <label htmlFor="amount">Montant à payer (en {currency}):</label>
                                <input type="number" id="amount" value={amount} onChange={e => setAmount(e.target.value)} min="100" required />

                                <label htmlFor="description">Description du paiement:</label>
                                <input type="text" id="description" value={description} onChange={e => setDescription(e.target.value)} required />
                                
                                <label htmlFor="customerFirstname">Prénom du client:</label>
                                <input type="text" id="customerFirstname" value={customerFirstname} onChange={e => setCustomerFirstname(e.target.value)} required />
                                
                                <label htmlFor="customerLastname">Nom du client:</label>
                                <input type="text" id="customerLastname" value={customerLastname} onChange={e => setCustomerLastname(e.target.value)} required />

                                <label htmlFor="customerEmail">Email du client:</label>
                                <input type="email" id="customerEmail" value={customerEmail} onChange={e => setCustomerEmail(e.target.value)} required />

                                <label htmlFor="customerPhone">Téléphone du client (ex: 90000000 pour BJ):</label>
                                <input type="text" id="customerPhone" value={customerPhone} onChange={e => setCustomerPhone(e.target.value)} required />
                                
                                <button type="submit">Payer</button>
                            </form>
                            {message && <div className={messageType}>{message}</div>}
                            <div id="fedapay-checkout-container"></div> {/* Conteneur optionnel pour FedaPay Checkout */}
                        </div>
                    );
                }

                ReactDOM.render(<App />, document.getElementById('root'));
            </script>
        </body>
        </html>
    `);
});

// Endpoint pour créer la transaction
app.post('/create-transaction', async (req, res) => {
    const {
        apiKeySecret, // Reçu du client UNIQUEMENT POUR CE TEST
        environment,
        amount,
        description,
        currency,
        customer
    } = req.body;

    if (!apiKeySecret) {
        return res.status(400).json({ message: "La clé API secrète est requise pour le backend." });
    }

    try {
        // Configure FedaPay SDK avec la clé secrète et l'environnement fournis dynamiquement pour ce test
        // DANS UNE VRAIE APP, la clé secrète vient de process.env et est configurée une seule fois au démarrage du serveur.
        FedaPay.setApiKey(apiKeySecret);
        FedaPay.setEnvironment(environment); // 'sandbox' ou 'live'

        console.log(\`Création de transaction en mode \${environment} pour \${amount} \${currency}\`);

        const transaction = await FedaPay.Transaction.create({
            description: description,
            amount: amount,
            currency: { iso: currency },
            customer: {
                firstname: customer.firstname,
                lastname: customer.lastname,
                email: customer.email,
                phone_number: {
                    number: customer.phone_number.number,
                    country: customer.phone_number.country
                }
            },
            // Callback URL : Où FedaPay redirigera après le paiement (ou enverra un POST)
            // Pour ce test simple, on se fie au onComplete de FedaPay.js
            // Dans une vraie app, il faudrait gérer ces callbacks sur votre serveur
            // callback_url: 'http://localhost:3000/fedapay-callback', 
        });
        
        // Générer un token pour le paiement
        const tokenObject = await transaction.generateToken();

        console.log('Transaction créée:', transaction.id);
        console.log('Token FedaPay:', tokenObject.token);
        // console.log('URL de paiement FedaPay:', tokenObject.url); // URL pour redirection directe

        // Renvoyer le token au client pour FedaPay Checkout (pop-up)
        res.json({ 
            token: tokenObject.token,
            transactionId: transaction.id
            // checkoutUrl: tokenObject.url // Si vous voulez rediriger au lieu d'utiliser FedaPay.js pop-up
        });

    } catch (error) {
        console.error('Erreur FedaPay:', error.message, error.response ? error.response.data : '');
        res.status(500).json({ 
            message: 'Erreur lors de la création de la transaction FedaPay.', 
            error: error.message,
            details: error.response ? error.response.data : null
        });
    }
});

// Endpoint de callback (optionnel pour ce test simple, mais crucial en production)
app.get('/fedapay-callback', async (req, res) => {
    const { id, status } = req.query; // FedaPay envoie l'ID de la transaction et son statut
    console.log(\`Callback FedaPay reçu pour transaction ID: \${id}, Statut: \${status}\`);

    if (!id) {
        return res.status(400).send('ID de transaction manquant dans le callback.');
    }
    // IMPORTANT: Récupérez votre clé API secrète depuis vos variables d'environnement ici
    // FedaPay.setApiKey(process.env.FEDAPAY_SECRET_KEY_SANDBOX); 
    // FedaPay.setEnvironment(process.env.FEDAPAY_ENVIRONMENT);
    
    // Pour ce test, on ne re-configure pas FedaPay car on ne sait pas quelle clé l'utilisateur a entré.
    // Dans une vraie app, les clés sont fixes côté serveur.

    try {
        // Vérifier la transaction pour confirmer son statut final
        // const transaction = await FedaPay.Transaction.retrieve(id);
        // console.log('Transaction récupérée depuis callback:', transaction.status, transaction.mode);
        // if (transaction.status === 'approved') {
        //     // Mettre à jour votre base de données, envoyer un email de confirmation, etc.
        //     res.send(\`Paiement pour la transaction \${id} confirmé et approuvé !\`);
        // } else {
        //     res.send(\`Paiement pour la transaction \${id} non approuvé. Statut : \${transaction.status}\`);
        // }
        res.send(\`Callback reçu pour transaction \${id}. Statut: \${status}. En production, vérifiez la transaction ici.\`);
    } catch (error) {
        console.error('Erreur lors de la récupération de la transaction via callback:', error);
        res.status(500).send('Erreur lors du traitement du callback FedaPay.');
    }
});


app.listen(PORT, () => {
    console.log(\`Serveur de test FedaPay démarré sur http://localhost:\${PORT}\`);
});
```

**Comment utiliser :**

1.  **Installer les dépendances :**
    Ouvrez votre terminal dans le dossier où vous avez sauvegardé `fedapay-test-app.js` et exécutez :
    ```bash
    npm init -y
    npm install express body-parser fedapay dotenv
    ```

2.  **Obtenir vos clés API FedaPay Sandbox :**
    *   Créez un compte sur [FedaPay](https://fedapay.com/).
    *   Accédez à votre tableau de bord et trouvez vos clés API pour l'environnement **Sandbox (Test)**. Vous aurez besoin de la "Clé Publique" et de la "Clé Secrète".

3.  **Exécuter l'application :**
    ```bash
    node fedapay-test-app.js
    ```

4.  **Ouvrir dans le navigateur :**
    Allez sur `http://localhost:3000`.

5.  **Remplir le formulaire :**
    *   Entrez votre clé API publique FedaPay (Sandbox).
    *   Entrez votre clé API secrète FedaPay (Sandbox).
    *   Laissez l'environnement sur "Sandbox".
    *   Entrez le montant (ex: 100 CFA), la description, et les informations client (celles-ci sont pour la transaction FedaPay, pas pour un "destinataire" externe).
    *   Cliquez sur "Payer".

6.  **Processus de Paiement FedaPay :**
    *   Le FedaPay Checkout (pop-up) devrait apparaître.
    *   FedaPay fournit des numéros de carte de test et des numéros de mobile money de test pour l'environnement Sandbox. Utilisez-les pour simuler un paiement. Vous les trouverez dans leur documentation.
        *   Exemple Mobile Money (Sandbox) : Opérateur MTN, numéro `90000000`, OTP `123456`.
        *   Exemple Carte (Sandbox) : Numéro `4242 4242 4242 4242`, date d'expiration future, CVC `123`.

7.  **Résultat :**
    *   Si le paiement est simulé avec succès, le callback `onComplete` de `FedaPay.checkout` sera exécuté, et vous verrez un message de succès sur la page.
    *   L'argent sera "crédité" sur votre compte FedaPay Sandbox. Vous pourrez le voir dans votre tableau de bord FedaPay (Sandbox).
    *   Regardez la console de votre terminal Node.js et la console de votre navigateur pour les logs.

Ce code est une base de test. Pour une application en production, vous devrez structurer votre code différemment (séparer le frontend et le backend, gérer les clés API de manière sécurisée avec des variables d'environnement uniquement sur le serveur, implémenter des webhooks robustes pour la confirmation des transactions, etc.).

C'est un bon point de départ pour comprendre le flux d'intégration de FedaPay !
