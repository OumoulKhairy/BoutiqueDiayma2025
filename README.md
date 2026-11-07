# Boutique Diayma

Application e-commerce ASP.NET Core MVC (ESP EHOD 2025) : liste de produits, panier, commande, choix de la langue.

> Dépôt d'origine : https://github.com/devehod/BoutiqueDiayma2025

---

## 1. Récupération et exécution

1. Visual Studio > **Cloner un dépôt** > coller `https://github.com/devehod/BoutiqueDiayma2025`.
2. Ouvrir `Diayma.sln`.
3. Fermer la fenêtre d'avertissements ⚠ (liste d'erreurs / avertissements), qui signale notamment que le framework cible `netcoreapp2.0` n'est plus pris en charge (NETSDK1138) et que les paquets 2.0.x ont des vulnérabilités connues. Ces avertissements ne bloquent pas la compilation.
4. Lancer avec **F5** (ou **Fn + F5**). Le navigateur s'ouvre sur `http://localhost:62929/` (profil projet) ou `http://localhost:60000/` (IIS Express).

---

## 2. Projets de la solution

La solution `Diayma.sln` contient **un seul projet** :

| Projet | Fichier | Type | Dossier |
|---|---|---|---|
| Diayma | `Diayma.csproj` | Application web ASP.NET Core MVC (`Microsoft.NET.Sdk.Web`) | `P2FixAnAppDotNetCode/` |

Le namespace racine est `P2FixAnAppDotNetCode` (nom historique du dossier), alors que l'assembly s'appelle `Diayma` (nom du `.csproj`).

Organisation du projet :

| Dossier | Contenu |
|---|---|
| `Controllers/` | `ProductController`, `CartController`, `OrderController`, `LanguageController` |
| `Components/` | View components `CartSummaryViewComponent`, `LanguageSelectorViewComponent` |
| `Models/` | `Product`, `Cart` / `CartLine`, `ICart`, `Order` |
| `Models/Services/` | `ProductService`, `OrderService`, `LanguageService` + interfaces |
| `Models/Repositories/` | `ProductRepository`, `OrderRepository` + interfaces (données en mémoire) |
| `Views/` | Vues Razor + `_Layout.cshtml` |
| `Resources/` | Fichiers `.resx` de traduction (`en`, `fr`) |

---

## 3. Version du SDK .NET

Dans `Diayma.csproj` :

```xml
<TargetFramework>netcoreapp2.0</TargetFramework>
...
<PackageReference Include="Microsoft.AspNetCore.All" Version="2.0.6" />
<PackageReference Include="Microsoft.Extensions.Localization" Version="2.1.1" />
```

- Framework cible : **.NET Core 2.0** (ASP.NET Core 2.0.6).
- SDK nécessaire : un **SDK .NET Core 2.1.1xx / 2.1.2xx** (le SDK 2.1.202 est le dernier de la série qui embarque le runtime 2.0).
- Pas de `global.json` : la version du SDK n'est pas figée, Visual Studio utilise le SDK le plus récent installé pour compiler, mais il faut le **runtime 2.0** pour exécuter l'application.
- Remarque : `.config/dotnet-tools.json` référence l'outil `dotnet-ef` 9.0.1, qui n'est pas utilisé par le projet (aucune base de données).

## 4. Installation du SDK

1. Télécharger le SDK sur https://dotnet.microsoft.com/download/dotnet/2.0 (ou la page des versions archivées .NET Core 2.1).
2. Installer, puis vérifier dans un terminal :

```
dotnet --list-sdks
dotnet --list-runtimes
```

`Microsoft.NETCore.App 2.0.x` et `Microsoft.AspNetCore.All 2.0.x` doivent apparaître dans la liste des runtimes.

---

## 5. Création de mon dépôt GitHub

1. Sur GitHub : **New repository** > nom `BoutiqueDiayma2025` > sans README (il existe déjà).
2. Dans le dossier cloné :

```
git remote rename origin upstream
git remote add origin https://github.com/<mon-compte>/BoutiqueDiayma2025.git
git push -u origin main
```

(`upstream` garde le lien vers le dépôt du professeur.)

---

## 6. Bugs trouvés

### Bug 1 : le total du panier ignore la quantité

- **Reproduction** : ajouter 2 fois le même produit (ex. Clé USB, 1 000). La ligne affiche une quantité de 2 et un sous-total de 2 000, mais le **Total** affiche 1 000. La moyenne est également fausse.
- **Cause** : `Models/Cart.cs`, ligne 66 :

```csharp
return GetCartLineList().Sum(x => x.Product.Price);
```

- **Correction** :

```csharp
public double GetTotalValue()
{
    return GetCartLineList().Sum(x => x.Product.Price * x.Quantity);
}

public double GetAverageValue()
{
    int totalQuantite = GetCartLineList().Sum(x => x.Quantity);
    return totalQuantite > 0 ? GetTotalValue() / totalQuantite : 0.0;
}
```

### Bug 2 : le stock n'est jamais décrémenté après une commande

- **Reproduction** : commander 3 « XIAOMI Redmi 13C » (stock 10), valider la commande, revenir à l'accueil : le stock affiche toujours **10**.
- **Cause** : `ProductRepository` est enregistré en `Transient` (`Startup.cs` ligne 32), donc une nouvelle instance est créée à chaque requête. Or son constructeur **réinitialise la liste statique** `_products` (`ProductRepository.cs` lignes 15-16) :

```csharp
public ProductRepository()
{
    _products = new List<Product>();   // efface les stocks mis à jour
    GenerateProductData();
}
```

- **Correction** : n'initialiser la liste qu'une seule fois.

```csharp
public ProductRepository()
{
    if (_products == null)
    {
        _products = new List<Product>();
        GenerateProductData();
    }
}
```

(Autre solution : `services.AddSingleton<IProductRepository, ProductRepository>();`.)

### Autres anomalies relevées

| Anomalie | Où | Effet |
|---|---|---|
| Choisir **Spanish** ne change rien | `LanguageService.SetCulture` renvoie `"es"`, mais `es` n'est pas dans `SupportedCultures` (`Startup.cs` l. 45-52) et aucun fichier `.es.resx` n'existe | L'interface reste en anglais |
| Le panier est **partagé entre tous les utilisateurs** | `services.AddSingleton<ICart, Cart>()` (`Startup.cs` l. 29) | Deux navigateurs différents voient le même panier. La session est configurée mais pas utilisée |
| Aucune vérification du stock | `CartController.AddToCart` / `Cart.AddItem` | On peut commander plus que le stock ; le stock devient négatif et, comme le test est `== 0`, le produit n'est pas retiré |
| Devise incohérente | `ToString("c")` dans les vues | Prix en FCFA affichés avec `¤` (culture `en` neutre) ou `€` (culture `fr`) |
| `NullReferenceException` possible | `Cart.FindProductInCartLines` : `.FirstOrDefault().Product` | Plantage si le produit n'est pas dans le panier |

---

## 7. Points d'arrêt

Tous les points d'arrêt demandés sont placés sur des **affectations dans les constructeurs** (injection de dépendances) :

| | Fichier | Ligne | Instruction |
|---|---|---|---|
| a | `Components/CartSummaryViewComponent.cs` | 12 | `_cart = cart as Cart;` |
| b | `Controllers/ProductController.cs` | 15 | `_productService = productService;` |
| c | `Controllers/OrderController.cs` | 17 | `_cart = pCart;` |
| d | `Controllers/CartController.cs` | 15 | `_cart = pCart;` |
| e | `Startup.cs` | 20 | `Configuration = configuration;` |

Pose : clic dans la marge grise ou **F9** sur la ligne, puis lancement avec **F5**.

---

## 8. Parcours d'exécution jusqu'à l'affichage des produits

Légende des modes :
- **F11 – Pas à pas détaillé** : entrer dans une méthode de **notre** code pour la suivre.
- **F10 – Pas à pas principal** : exécuter une ligne sans entrer dans l'appel (affectations, appels au framework, lambdas LINQ).
- **Shift+F11 – Pas à pas sortant** : terminer la méthode en cours et revenir à l'appelant.
- **F5 – Continuer** : aller jusqu'au point d'arrêt suivant.

> Avec l'option « Uniquement mon code » (activée par défaut), le débogueur ne s'arrête pas dans le code du framework (`Microsoft.AspNetCore.*`). Les méthodes du framework sont donc **traversées** mais pas affichées pas à pas.

### Étape 1 – Démarrage de l'application

| # | Namespace | Classe | Méthode | Mode |
|---|---|---|---|---|
| 1 | `P2FixAnAppDotNetCode` | `Program` | `Main(string[] args)` | F11 sur `BuildWebHost(args).Run()` |
| 2 | `P2FixAnAppDotNetCode` | `Program` | `BuildWebHost(string[] args)` | F10 |
| 3 | `Microsoft.AspNetCore` | `WebHost` | `CreateDefaultBuilder(args)` | (framework, traversé) |
| 4 | `Microsoft.AspNetCore.Hosting` | `WebHostBuilderExtensions` / `WebHostBuilder` | `UseStartup<Startup>()`, `Build()` | (framework, traversé) |
| 5 | `P2FixAnAppDotNetCode` | `Startup` | **constructeur** `Startup(IConfiguration)` → **arrêt e) ligne 20** | F10 |
| 6 | `P2FixAnAppDotNetCode` | `Startup` | `ConfigureServices(IServiceCollection)` : enregistrement des services (`ICart` singleton, services et repositories transient, localisation, MVC) | F10 ligne par ligne |
| 7 | `P2FixAnAppDotNetCode` | `Startup` | `Configure(IApplicationBuilder, IHostingEnvironment)` : `UseStaticFiles`, `UseRequestLocalization`, `UseSession`, `UseMvc` avec la route `{controller=Product}/{action=Index}/{id?}` | F10, puis F5 |
| 8 | `Microsoft.AspNetCore.Hosting` | `WebHost` | `Run()` : Kestrel démarre et le navigateur envoie `GET /` | F5 |

### Étape 2 – Routage et création du contrôleur

La route par défaut envoie `/` vers `Product/Index`. Le conteneur d'injection de dépendances (`Microsoft.Extensions.DependencyInjection`) construit d'abord les dépendances, **puis** le contrôleur :

| # | Namespace | Classe | Méthode |
|---|---|---|---|
| 9 | `P2FixAnAppDotNetCode.Models.Repositories` | `ProductRepository` | constructeur → `GenerateProductData()` (9 produits) |
| 10 | `P2FixAnAppDotNetCode.Models.Repositories` | `OrderRepository` | constructeur |
| 11 | `P2FixAnAppDotNetCode.Models.Services` | `ProductService` | constructeur `ProductService(IProductRepository, IOrderRepository)` |
| 12 | `P2FixAnAppDotNetCode.Models.Services` | `LanguageService` | constructeur par défaut (singleton, une seule fois) |
| 13 | `P2FixAnAppDotNetCode.Controllers` | `ProductController` | **constructeur → arrêt b) ligne 15** |

Mode : **F10** sur les lignes 15 et 16, puis **F10** pour sortir du constructeur.

> Les constructeurs 9 à 12 n'ont pas de point d'arrêt : on ne s'y arrête pas, mais ils sont exécutés **avant** la ligne 15 (visible dans la fenêtre **Pile des appels**).

### Étape 3 – Action `Index` et récupération des produits

| # | Namespace | Classe | Méthode | Mode |
|---|---|---|---|---|
| 14 | `P2FixAnAppDotNetCode.Controllers` | `ProductController` | `Index()` | F11 sur la ligne 21 |
| 15 | `P2FixAnAppDotNetCode.Models.Services` | `ProductService` | `GetAllProducts()` | F11 sur la ligne 29 |
| 16 | `P2FixAnAppDotNetCode.Models.Repositories` | `ProductRepository` | `GetAllProducts()` : filtre `Stock > 0`, tri par `Name`, `ToArray()` | F10 (sinon F11 entre 9 fois dans la lambda `p => p.Stock > 0`) |
| 17 | `System.Linq` | `Enumerable` | `Where`, `OrderBy`, `ToList`, `ToArray` | (framework, traversé) |
| 18 | `P2FixAnAppDotNetCode.Models.Repositories` | `ProductRepository` | retour du tableau | **Shift+F11** |
| 19 | `P2FixAnAppDotNetCode.Models.Services` | `ProductService` | `.ToList()` puis retour de `List<Product>` | **Shift+F11** |
| 20 | `P2FixAnAppDotNetCode.Controllers` | `ProductController` | `return View(products);` | F10 |
| 21 | `Microsoft.AspNetCore.Mvc` | `Controller` | `View(object model)` → `ViewResult` | (framework, traversé) |

### Étape 4 – Rendu de la page (vue + layout)

Le moteur Razor exécute d'abord la vue, puis le layout qui l'entoure :

| # | Élément | Détail |
|---|---|---|
| 22 | `Views/_ViewStart.cshtml` | Définit `_Layout` comme layout |
| 23 | `Views/Product/Index.cshtml` | Boucle `foreach (Product p in Model)` : génère le tableau (nom, description, prix, stock, bouton « Add to cart ») |
| 24 | `Views/Shared/_Layout.cshtml` | `@await Component.InvokeAsync("CartSummary")` |
| 25 | `P2FixAnAppDotNetCode.Models` → `Cart` | constructeur (singleton, **seulement à la 1re requête**) : initialisation de `_lignesPanier` |
| 26 | `P2FixAnAppDotNetCode.Components` → `CartSummaryViewComponent` | **constructeur → arrêt a) ligne 12**, mode F10 |
| 27 | `P2FixAnAppDotNetCode.Components` → `CartSummaryViewComponent` | `Invoke()` → `View(_cart)` (F11 pour y entrer, Shift+F11 pour en sortir) |
| 28 | `Views/Shared/Components/CartSummary/Default.cshtml` | `Model.Lines.Count()` = 0 → seul le bouton panier s'affiche |
| 29 | `Views/Shared/_Layout.cshtml` | `@RenderBody()` insère le tableau produit à l'étape 23 |
| 30 | `P2FixAnAppDotNetCode.Components` → `LanguageSelectorViewComponent` | `Invoke(ILanguageService languageService)` → `View(languageService)` |
| 31 | `Views/Shared/Components/LanguageSelector/Default.cshtml` | Liste déroulante English / French / Spanish |
| 32 | Navigateur | La page HTML complète est envoyée : **les produits s'affichent** |

Ensuite, **F5** pour continuer.

### Points d'arrêt **non** atteints à l'affichage de l'accueil

- **c) OrderController ligne 17** : atteint seulement en cliquant sur **Checkout** (`/Order/Index`).
- **d) CartController ligne 15** : atteint seulement en cliquant sur **Add to cart** ou sur l'icône panier.

### Résumé des namespaces visités

| Namespace | Classes |
|---|---|
| `P2FixAnAppDotNetCode` | `Program`, `Startup` |
| `P2FixAnAppDotNetCode.Controllers` | `ProductController` |
| `P2FixAnAppDotNetCode.Models` | `Product`, `Cart` |
| `P2FixAnAppDotNetCode.Models.Services` | `ProductService`, `LanguageService` |
| `P2FixAnAppDotNetCode.Models.Repositories` | `ProductRepository`, `OrderRepository` |
| `P2FixAnAppDotNetCode.Components` | `CartSummaryViewComponent`, `LanguageSelectorViewComponent` |
| Framework | `Microsoft.AspNetCore`, `Microsoft.AspNetCore.Hosting`, `Microsoft.AspNetCore.Mvc`, `Microsoft.Extensions.DependencyInjection`, `System.Linq` |

---

## 9. Déploiement en exécutable Windows

Depuis le dossier `P2FixAnAppDotNetCode` :

```
dotnet publish -c Release -r win-x64 --self-contained true -o publish
```

Ou dans Visual Studio : clic droit sur le projet > **Publier** > profil `FolderProfile` (modifier le dossier cible `F:\Diayma`, qui n'existe peut-être pas, par un dossier local).

Résultat : `publish\Diayma.exe` (+ `wwwroot`, DLL, vues précompilées).

Lancement : double-cliquer sur `Diayma.exe`, puis ouvrir **http://localhost:5000** dans le navigateur.

> - Le mode `--self-contained` embarque le runtime : pas besoin d'installer .NET sur la machine cible.
> - L'option `PublishSingleFile` (un seul `.exe`) **n'existe pas en .NET Core 2.0** (introduite en .NET Core 3.0) : il faut distribuer **tout le dossier**.

## 10. Lien de téléchargement

Archive ZIP du dossier `publish` : **[lien Google Drive à ajouter]**

(Partager le **dossier complet zippé**, pas seulement le `.exe`, qui ne fonctionne pas seul.)

---

## 11. Commits

| Commit | Description |
|---|---|
| 1 | Correction du calcul du total et de la moyenne du panier |
| 2 | Correction de la réinitialisation du stock dans `ProductRepository` |
| 3 | Ajout de la langue wolof (`wo`) en conservant le français |
