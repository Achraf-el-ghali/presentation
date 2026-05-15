# 📋 RÉVISION SOUTENANCE PFA — GearOil

---

## 🏗️ ARCHITECTURE GLOBALE

```
Angular 21 → API Gateway Ocelot (.NET 9) → 6 Microservices → Apache Kafka
```

| Service | Techno | Base de données |
|---|---|---|
| Identity & Gateway | .NET 9 + Ocelot | SQL Server |
| Commandes & Paiement | Spring Boot 3.2.5 | MySQL |
| Catalogue & Recommandation | Symfony 7 + API Platform | MongoDB |
| Stock | Symfony 7 + API Platform | MySQL |
| Retours | Symfony 7 | MongoDB |
| Notifications | Spring Boot | MySQL |

---

---

## 👤 MS 1 — USER SERVICE (.NET 9)

### Rôle :
Authentification, gestion des utilisateurs, JWT, SignalR Hub

### Modèles :
- **User** (abstrait) : Id, Nom, Prenom, Username, Email, PasswordHash, Role, CreatedAt
- **Admin** : hérite de User (rien de plus)
- **Client** : hérite de User + champ `Bloque` (bool)
- **UserExternalLogin** : Provider, ProviderUserId, UserId (pour Google/Facebook)

### Comment on l'a fait :
- **Héritage TPH** : une seule table avec discriminateur `UserType` ("Admin" ou "Client")
- **Index Unique** : Email et Username indexés → pas de doublons + recherche rapide
- **Cascade Delete** : supprimer un User → supprime ses ExternalLogins
- **Migrations auto** : `db.Database.Migrate()` au démarrage
- **Password** : ASP.NET Identity hash avec PBKDF2 + salt automatiquement

### JWT (Symétrique HMAC-SHA256) :
- User Service GÉNÈRE le token avec une clé secrète
- API Gateway VÉRIFIE avec la MÊME clé secrète
- Access Token = 15 min / Refresh Token = 7 jours
- Le token contient : userId, role, email, expiration

### Login Google :
1. Créer projet sur Google Cloud Console → récupérer Client ID + Client Secret
2. Angular redirige vers Google → l'utilisateur se connecte
3. Google renvoie un code → Angular envoie au backend
4. Backend échange code + Client Secret auprès de Google → reçoit email + ProviderUserId
5. Cherche en base : ProviderUserId existe ? → OUI : connecte / NON : crée User + ExternalLogin
6. Génère notre JWT

### SignalR :
- Bibliothèque .NET = Hub de communication temps réel via WebSocket
- Les clients Angular se connectent au Hub
- Le serveur pousse les notifications instantanément

---

---

## 💳 MS 2 — COMMANDES & PAIEMENT (Spring Boot 3.2.5)

### Rôle :
Créer commandes, gérer paiements Stripe, livraison, vidange

### Entités :
- **Order** : orderNumber, clientId, totalAmount, status (PENDING/PAID/FAILED/CANCELLED), paymentIntentId, clientSecret, items
- **OrderItem** : sku, price, quantity, type (DELIVERY/VIDANGE), vidangeType, scheduledAt
- **Payment** : orderId, amount, paymentMethod, status, transactionId
- **OutboxEvent** : id (UUID), eventType, payload, status (PENDING/SENT), sentAt

### Statuts d'une commande :
```
PENDING → PAID (Stripe confirme)
PENDING → FAILED (carte refusée)
PENDING → CANCELLED (client annule)
```

### Stripe — Le flux complet :
1. Client commande → commande PENDING en base
2. Backend crée PaymentIntent chez Stripe (avec metadata orderId)
3. Stripe retourne clientSecret → envoyé au frontend
4. Frontend affiche formulaire Stripe → client entre sa carte → va DIRECTEMENT chez Stripe
5. Stripe vérifie avec la banque
6. Stripe envoie WEBHOOK à notre endpoint `/api/payments/webhook`
7. Backend vérifie la signature (Webhook Secret)
8. `markAsPaid()` → PENDING → PAID + écrit Outbox (même transaction)
9. OutboxWorker → envoie "order.paid" à Kafka
10. Kafka → Stock décrémente, Livraison créée, Retours cache, SMS envoyé

### Sécurité Stripe :
- Les numéros de carte ne passent JAMAIS par notre serveur
- HTTPS partout
- Webhook signé → `Webhook.constructEvent(payload, signature, secret)`
- Idempotence → `findByTransactionId()` empêche double paiement

### Pattern Outbox :
- Au lieu d'envoyer à Kafka directement, on ÉCRIT dans table `outbox_events` (même transaction)
- `OutboxWorker` tourne toutes les 10s : lit PENDING → envoie à Kafka → marque SENT
- Garantie : rien n'est perdu même si Kafka est down

---

---

## 📦 MS 3 — STOCK (Symfony 7 + MySQL)

### Rôle :
Gérer les quantités, lots FIFO, mouvements, promotions

### Entités :
- **Stock** : sku (unique), quantity, price, isActive, promotions
- **StockLot** : sku, quantityInitial, quantityRemaining, purchasePrice, sellingPrice, dateEntry
- **StockMovement** : sku, lotId, type (IN/OUT), quantity, source, createdAt
- **Promotion** : description, type (percentage/fixed), value, stocks (ManyToMany)
- **ProcessedEvent** : eventId, serviceName (pour idempotence)

### FIFO (First In, First Out) :
- Quand on décrémente, on prend dans les lots les plus ANCIENS d'abord
- Lot 1 (01/05) → Lot 2 (10/05) → on vide Lot 1 avant Lot 2

### Kafka Consumer — Événements écoutés :
| Événement | Action |
|---|---|
| `product-created` | Crée Stock avec quantité 0 |
| `product-deleted` | Désactive (isActive = false) |
| `product-updated` | Met à jour le statut |
| `stock-added` | addStock() — crée lot + incrémente |
| `stock-decrease-requested` | decreaseStock() — décrémente en FIFO |

→ Après chaque traitement, publie `stock-updated` sur Kafka

### Idempotence :
- Table `ProcessedEvent` avec eventId + serviceName
- Avant de traiter : `isProcessed(eventId, 'stock-service')` → si OUI → ignore
- Après traitement : `recordProcessedEvent(eventId)` dans la MÊME transaction

### Verrouillage Pessimiste :
- `findOneBySkuWithLock()` → `SELECT ... FOR UPDATE`
- Bloque la ligne en base → le 2ème attend
- Si stock insuffisant → exception + rollback

### Promotions :
- `getFinalPrice()` parcourt les promotions du produit
- Pourcentage : prix - (prix × value / 100)
- Fixe : prix - value
- `max(0.0, finalPrice)` → jamais négatif

### Notifications :
| Événement | Message |
|---|---|
| Nouveau produit | "Nouveau produit SKU:X ajouté" |
| Retour en stock | "L'article X est de nouveau en stock !" |
| Stock faible (<10) | "⚠️ STOCK FAIBLE" |
| Rupture (=0) | "🛑 RUPTURE DE STOCK" |

---

---

## 🔄 MS 4 — RETOURS (Symfony 7 + MongoDB)

### Rôle :
Gérer les demandes de retour produit

### Workflow (State Machine Symfony) :
```
PENDING → APPROVED → REFUNDED
PENDING → REJECTED
```

### Entité ProductReturn (MongoDB) :
- orderReference, sku, quantity, status, condition (new/used/damaged), reason, imageData
- customerName, createdAt, inspectedAt, approvedAt, refundedAt

### Comment ça marche :
1. Client demande retour (POST /api/returns)
2. Service vérifie : commande payée ? livrée ? quantité valide ?
3. Si OK → crée retour PENDING
4. Admin approuve ou rejette via Workflow
5. Si APPROVED → événement Kafka `return_status_updates` → Stock réincrémente
6. Si REFUNDED → événement Kafka `payment_refund_commands` → Spring Boot rembourse via Stripe

### MongoDB comme cache :
- 3 Workers Kafka écrivent les commandes en local MongoDB :
  - `worker_orders` → écoute nouvelles commandes
  - `worker_payments` → écoute paiements confirmés
  - `worker_deliveries` → écoute livraisons
- Quand un retour est demandé → le service vérifie dans SON MongoDB (pas d'appel externe)
- Avantage : autonome, rapide, découplé

---

---

## 📱 MS 5 — NOTIFICATIONS (SignalR + Twilio)

### Flux complet :
```
Symfony → POST HTTPS → Spring Boot (signe avec clé privée RSA + stocke en base)
→ POST signé → .NET User Service (vérifie avec clé publique)
→ SignalR Hub (WebSocket) → Angular (notification instantanée)
```

### SMS via Twilio :
- Quand livreur/technicien est à moins de 2 km du client
- `NotificationService.sendProximitySms()` → Twilio envoie un vrai SMS

### Sécurité (RSA Asymétrique) :
- Spring Boot signe avec clé PRIVÉE
- .NET vérifie avec clé PUBLIQUE
- Si signature invalide → rejeté (pas un pirate)

---

---

## 🔒 SÉCURITÉ — Résumé

| Mécanisme | Type | Où |
|---|---|---|
| JWT (Auth) | Symétrique (HMAC-SHA256) | User Service ↔ Gateway |
| Notifications | Asymétrique (RSA) | Spring Boot → .NET |
| Webhook Stripe | Symétrique (HMAC) | Stripe → Spring Boot |
| HTTPS/TLS | Chiffrement | Partout |
| Stripe PCI-DSS | Externe | Cartes jamais chez nous |

---

---

## ⚡ BESOINS NON FONCTIONNELS

| Axe | Comment on l'a fait |
|---|---|
| **Performance** | Redis cache, Indexation BDD, Pagination (API Platform), Lazy Loading Angular |
| **Fiabilité** | Outbox (at-least-once), Idempotence (ProcessedEvent), ACID (@Transactional), Verrouillage (FOR UPDATE) |
| **Scalabilité** | Microservices indépendants, Kafka asynchrone, API Gateway Ocelot |
| **Sécurité** | JWT (access+refresh), HTTPS, Stripe PCI-DSS, Webhook signé |
| **Maintenabilité** | SOLID (ISP), Design Patterns (Facade, Decorator), Code modulaire |
| **Portabilité** | Docker, Variables .env, Réseau gearoil_network |
| **Accessibilité** | Interfaces par rôle (JWT claims), Navigation simplifiée, UX fluide |
| **IA/UX** | Recombee (recommandations), Notifications temps réel, Personnalisation |

---

---

## 🎯 PHRASES CLÉS pour le jury :

1. "Nous avons utilisé le pattern Outbox pour garantir la livraison des événements même si Kafka est indisponible."
2. "Chaque service a sa propre base de données — c'est le principe Database per Service."
3. "L'idempotence nous protège contre les doublons en cas de retry Kafka."
4. "SignalR maintient une connexion WebSocket permanente pour les notifications temps réel."
5. "Le verrouillage pessimiste empêche les conflits de concurrence sur le stock."
6. "Nous avons séparé nos interfaces selon le principe ISP — chaque controller ne dépend que de ce qu'il utilise."
7. "Les numéros de carte ne passent jamais par notre serveur — Stripe gère tout côté paiement."

---

Bonne chance pour la soutenance ! 💪🚀
