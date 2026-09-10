# gold-futures-var-es-backtesting

Projet de gestion des risques financiers (UNamur) réalisé sous Gretl. Estimation de la VaR et de l'Expected Shortfall des Gold Futures (GC=F) par simulation historique, EWMA, ARCH, GARCH et GJR-GARCH (lois normale et de Student), puis backtesting hors échantillon (test de Kupiec, diagnostic de l'ES et test de robustesse).



# VaR et Expected Shortfall des Gold Futures : comparaison de modèles et backtesting hors échantillon
 
Ce dépôt contient le script Gretl développé dans le cadre du projet du cours de Gestion du risque financier (ELFIM402) à l'Université de Namur, sous la direction du Prof. Jean-Yves Gnabo. Le projet a été réalisé en groupe avec Antoine Arts et Mélissa Klica, le script ayant été développé par Lorys Massaux.
 
## Objectif
 
Nous mesurons le risque de marché de l'or à travers deux indicateurs, la Value at Risk (VaR) et l'Expected Shortfall (ES), aux seuils de 1 %, 5 % et 10 %. Plusieurs modèles sont estimés puis confrontés aux pertes effectivement observées sur une période hors échantillon, afin de recommander le modèle le plus adapté à l'actif étudié.
 
## Données
 
L'actif étudié est le contrat à terme sur l'or (Gold Futures, GC=F) négocié sur le COMEX. Les prix de clôture ajustés ont été récupérés depuis Yahoo Finance à l'aide du package Python `yfinance` et ne sont pas inclus dans ce dépôt.
 
L'échantillon couvre 750 jours de cotation, du 4 avril 2023 au 16 février 2026. Les valeurs manquantes sont comblées par forward fill, en supposant qu'aucune transaction n'a eu lieu durant ces périodes. Les rendements sont calculés de manière logarithmique et les pertes sont définies selon la convention Lt = −rt, de sorte qu'un rendement négatif correspond à une perte positive.
 
L'échantillon est scindé en deux. Les 500 premières observations servent à l'estimation des modèles, tandis que les 250 dernières sont utilisées pour les prévisions et le backtesting hors échantillon.
 
## Modèles estimés
 
| Modèle | Volatilité | Loi des innovations | Estimation |
|---|---|---|---|
| Simulation historique | aucune modélisation, fenêtre glissante de 250 jours | empirique | calcul direct |
| EWMA | λ = 0,94, variance initiale égale à la variance des 500 premiers rendements | normale | calcul récursif |
| ARCH(2) | chocs des deux périodes précédentes | normale | `garch 0 2` |
| GARCH(1,1) | chocs et variance de la période précédente | normale | `garch 1 1` |
| GARCH(1,1)-Student | idem | Student | package `gig` |
| GJR-GARCH | ajout d'un effet d'asymétrie des chocs | normale | package `gig` |
| GJR-GARCH-Student | idem | Student | package `gig` |
 
Le modèle ARCH(1) initialement prévu n'ayant pas convergé sur notre échantillon, il a été remplacé par un ARCH(2).
 
Pour les modèles paramétriques, les coefficients sont estimés sur les 500 premières observations. La variance conditionnelle des 250 observations hors échantillon est ensuite prévue pas à pas, en appliquant récursivement l'équation de variance avec les coefficients estimés et les résidus observés. La VaR est obtenue par VaR = −μ + z·σ, le terme μ étant omis pour l'EWMA en supposant une moyenne des rendements journaliers nulle. Pour la simulation historique, la VaR correspond au quantile empirique des pertes de la fenêtre glissante.
 
Pour l'ensemble des modèles, l'ES d'un jour donné est calculée comme la moyenne des pertes de la fenêtre des 250 jours précédents qui dépassent la VaR prévue par le modèle pour ce jour.
 
Les critères AIC et BIC de l'ARCH(2), du GARCH(1,1) et du GARCH(2,2) sont également comparés afin d'évaluer leur qualité d'ajustement.
 
## Backtesting
 
Le backtesting est réalisé sur les 250 observations hors échantillon, pour chaque modèle et chaque seuil.
 
**Exceptions de VaR.** Une exception est enregistrée lorsque la perte observée dépasse la VaR prévue. Le nombre d'exceptions est comparé au nombre théoriquement attendu, soit 2,5 à 1 %, 12,5 à 5 % et 25 à 10 %.
 
**Qualité de couverture.** Le test de Kupiec (1995) vérifie si le taux d'exceptions observé est statistiquement égal au taux attendu, sa statistique suivant une loi du χ² à un degré de liberté.
 
**Diagnostic de l'ES.** L'ES observée, qui correspond à la moyenne des pertes enregistrées les jours d'exception, est comparée à la moyenne des ES prédites.
 
**Test de robustesse.** La période hors échantillon est scindée en deux sous-périodes de 125 observations, sur lesquelles le nombre d'exceptions, leur taux et la p-valeur du test de Kupiec sont recalculés.
 
## Structure du script
 
Le script `script_gold.inp` est organisé en sections successives.
 
| Section | Contenu |
|---|---|
| Prérequis | Chargement du package `gig`, forward fill, calcul des rendements et des pertes |
| Rolling window | Simulation historique |
| EWMA | Calcul récursif de la variance et VaR paramétrique |
| ARCH | Estimation de l'ARCH(2) et prévision hors échantillon |
| GARCH(1,1) | Estimation et prévision hors échantillon |
| Comparaison AIC | Critères AIC et BIC de l'ARCH(2), du GARCH(1,1) et du GARCH(2,2) |
| Modèles supplémentaires | GARCH-Student, GJR-GARCH et GJR-GARCH-Student |
| Génération des graphiques | Pertes et VaR, VaR et ES, volatilités conditionnelles, violations par modèle |
| Tableaux de synthèse | Synthèse au seuil de 1 % et synthèse de la robustesse |
 
Chaque bloc de modèle suit la même séquence, à savoir le calcul de la VaR et de l'ES, le backtesting avec le test de Kupiec, le diagnostic de l'ES puis le test de robustesse. Pour la simulation historique et l'EWMA, les séries de VaR et d'ES des trois seuils sont créées dynamiquement au sein d'une boucle, grâce à `sprintf` et à l'opérateur `@`.
 
## Principaux résultats
 
Le GARCH(1,1) présente les critères AIC et BIC les plus faibles parmi les spécifications comparées. Au seuil de 1 %, seuls la simulation historique et le GARCH-Student passent le test de Kupiec, tandis que l'ARCH(2) sous-estime nettement le risque avec 19 exceptions. Pour l'ensemble des modèles, l'ES observée reste supérieure à l'ES prédite, ce qui traduit une sous-estimation de la sévérité des pertes extrêmes. Le rapport recommande le GARCH(1,1)-Student comme modèle principal et la simulation historique comme modèle de secours (voir toutefois la section suivante).
 
## Utilisation
 
Le script nécessite Gretl ainsi que le package `gig`, installable depuis le serveur de paquets de Gretl.
 
Il s'exécute sur un jeu de données Gretl de séries temporelles journalières (5 jours par semaine) contenant la série `AdjClose`, dont les observations 1 à 500 correspondent à l'échantillon d'estimation et les observations 501 à 750 à l'échantillon hors échantillon. Les graphiques étant générés avec l'option `--output=display`, le script est prévu pour être exécuté depuis l'interface graphique de Gretl.
 
## Références
 
- Bollerslev, T. (1986). Generalized autoregressive conditional heteroskedasticity. *Journal of Econometrics*, 31(3), 307-327.
- Engle, R. F. (1982). Autoregressive conditional heteroskedasticity with estimates of the variance of United Kingdom inflation. *Econometrica*, 50(4), 987-1008.
- Glosten, L. R., Jagannathan, R., & Runkle, D. E. (1993). On the relation between the expected value and the volatility of the nominal excess return on stocks. *The Journal of Finance*, 48(5), 1779-1801.
- Kupiec, P. H. (1995). Techniques for verifying the accuracy of risk measurement models. *The Journal of Derivatives*, 3(2), 73-84.
- Lucchetti, R., & Balietti, S. *gig, An assortment of univariate GARCH models* [package Gretl].
