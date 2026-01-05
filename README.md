# Dashboard-Livraisons-CCV
import React, { useState, useMemo } from 'react';
import { Upload, Package, Clock, AlertTriangle, TrendingUp, Download, RefreshCw, Filter, FileDown } from 'lucide-react';
import * as XLSX from 'xlsx';

const LogisticsDashboard = () => {
  const [orders, setOrders] = useState([]);
  const [loading, setLoading] = useState(false);
  const [selectedHub, setSelectedHub] = useState('TOUS');
  const [dateCommandeStart, setDateCommandeStart] = useState('');
  const [dateCommandeEnd, setDateCommandeEnd] = useState('');
  const [dateChargementStart, setDateChargementStart] = useState('');
  const [dateChargementEnd, setDateChargementEnd] = useState('');
  const [showFilters, setShowFilters] = useState(false);
  
  // Filtres temporaires (avant application)
  const [tempSelectedHub, setTempSelectedHub] = useState('TOUS');
  const [tempDateCommandeStart, setTempDateCommandeStart] = useState('');
  const [tempDateCommandeEnd, setTempDateCommandeEnd] = useState('');
  const [tempDateChargementStart, setTempDateChargementStart] = useState('');
  const [tempDateChargementEnd, setTempDateChargementEnd] = useState('');

  // Fonction de conversion des dates Excel vers format ISO
  const convertExcelDate = (excelDate) => {
    if (!excelDate) return '';
    
    if (typeof excelDate === 'string') {
      if (excelDate.includes('/')) {
        const parts = excelDate.trim().split('/');
        if (parts.length === 3) {
          const day = parts[0].padStart(2, '0');
          const month = parts[1].padStart(2, '0');
          const year = parts[2].length === 2 ? '20' + parts[2] : parts[2];
          return `${year}-${month}-${day}`;
        }
      }
      if (excelDate.includes('-')) {
        return excelDate;
      }
    }
    
    if (typeof excelDate === 'number') {
      const excelEpoch = new Date(1900, 0, 1);
      const days = excelDate - 2;
      const date = new Date(excelEpoch.getTime() + days * 24 * 60 * 60 * 1000);
      
      const year = date.getFullYear();
      const month = String(date.getMonth() + 1).padStart(2, '0');
      const day = String(date.getDate()).padStart(2, '0');
      return `${year}-${month}-${day}`;
    }
    
    if (excelDate instanceof Date) {
      const year = excelDate.getFullYear();
      const month = String(excelDate.getMonth() + 1).padStart(2, '0');
      const day = String(excelDate.getDate()).padStart(2, '0');
      return `${year}-${month}-${day}`;
    }
    
    return '';
  };

  // Fonction pour formater les dates en JJ/MM/AAAA pour l'affichage
  const formatDateDisplay = (isoDate) => {
    if (!isoDate) return '-';
    try {
      const [year, month, day] = isoDate.split('-');
      return `${day}/${month}/${year}`;
    } catch {
      return isoDate;
    }
  };

  // Fonction d'import Excel
  const handleFileUpload = (e) => {
    const file = e.target.files[0];
    if (!file) return;

    setLoading(true);
    const reader = new FileReader();

    reader.onload = (evt) => {
      try {
        const data = evt.target.result;
        const workbook = XLSX.read(data, { type: 'binary', cellDates: false, cellNF: false, cellText: false });
        const sheetName = workbook.SheetNames[0];
        const worksheet = workbook.Sheets[sheetName];
        
        const jsonData = XLSX.utils.sheet_to_json(worksheet, { raw: true, defval: '' });

        const processedOrders = jsonData.map((row, index) => ({
          commande: row['Commande'] || row['commande'] || `CMD-${index + 1}`,
          dateCommande: convertExcelDate(row['Date'] || row['date']),
          client: row['Client'] || row['client'] || 'Non spécifié',
          cpDestination: row['CP Destination'] || row['cp destination'] || '',
          villeDestination: row['Ville Destination'] || row['ville destination'] || '',
          dateChargementMMK: convertExcelDate(row['date Chargement mmk'] || row['date chargement mmk'] || row['Date Chargement mmk']),
          code: row['Code'] || row['code'] || '',
          codeHubInjection: row['CODE HUB INJECTION'] || row['code hub injection'] || 'ANZ',
          dateCreationAnnonceCCV: convertExcelDate(row['DATE CREATION ANNONCE CCV'] || row['date creation annonce ccv']),
          dateLivraisonClient: convertExcelDate(row['DATE LIVRAISON CLIENT'] || row['date livraison client']),
          dateDernierEvenementCCV: convertExcelDate(row['DATE DERNIER EVENEMENT CCV'] || row['date dernier evenement ccv']),
          dernierEvenementCCV: row['DERNIER EVENEMENT CCV'] || row['dernier evenement ccv'] || '',
          datePremierRDVCCV: convertExcelDate(row['DATE DU PREMIER RDV CCV'] || row['date du premier rdv ccv']),
          dateReceptionHubInjection: convertExcelDate(row['DATE RECEPTION HUB INJECTION'] || row['date reception hub injection']),
        }));

        setOrders(processedOrders);
      } catch (error) {
        alert('Erreur lors de la lecture du fichier. Vérifiez le format.');
        console.error('Erreur import:', error);
      } finally {
        setLoading(false);
      }
    };

    reader.readAsBinaryString(file);
  };

  // Calcul des jours entre deux dates
  const calculateDays = (date1, date2) => {
    if (!date1 || !date2) return null;
    if (date1 === '-' || date2 === '-') return null;
    
    try {
      const d1 = new Date(date1);
      const d2 = new Date(date2);
      
      if (isNaN(d1.getTime()) || isNaN(d2.getTime())) return null;
      
      const diffTime = d2 - d1;
      const days = Math.round(diffTime / (1000 * 60 * 60 * 24));
      
      return days;
    } catch {
      return null;
    }
  };

  // Filtrage des commandes
  const filteredOrders = useMemo(() => {
    return orders.filter(order => {
      const hubMatch = selectedHub === 'TOUS' || order.codeHubInjection === selectedHub;
      
      const dateCommandeMatch = (!dateCommandeStart || order.dateCommande >= dateCommandeStart) &&
                                (!dateCommandeEnd || order.dateCommande <= dateCommandeEnd);
      
      const dateChargementMatch = (!dateChargementStart || order.dateChargementMMK >= dateChargementStart) &&
                                  (!dateChargementEnd || order.dateChargementMMK <= dateChargementEnd);
      
      return hubMatch && dateCommandeMatch && dateChargementMatch;
    });
  }, [orders, selectedHub, dateCommandeStart, dateCommandeEnd, dateChargementStart, dateChargementEnd]);

  // Fonction helper pour vérifier si une date est vide
  const isDateEmpty = (date) => {
    if (!date) return true;
    if (date === '-') return true;
    if (typeof date === 'string' && date.trim() === '') return true;
    return false;
  };

  // ATTENTE HUB : Aucune des 3 dates clés (réception hub, RDV, livraison)
  const getCommandesAttenteHub = (ordersList) => {
    return ordersList.filter(o => {
      // Une commande est en "Attente HUB" si elle n'a AUCUNE de ces 3 dates :
      const pasDeReceptionHub = isDateEmpty(o.dateReceptionHubInjection);
      const pasDeRDV = isDateEmpty(o.datePremierRDVCCV);
      const pasDeLivraison = isDateEmpty(o.dateLivraisonClient);
      
      // Les 3 doivent être vides (ET logique)
      return pasDeReceptionHub && pasDeRDV && pasDeLivraison;
    });
  };

  // Fonction helper pour vérifier si une date existe
  const hasDate = (date) => {
    return !isDateEmpty(date);
  };

  // SANS RDV : A une réception hub MAIS pas de RDV
  // SANS RDV : A une réception hub MAIS pas de RDV
  const getCommandesSansRDV = (ordersList) => {
    return ordersList.filter(o => {
      const aReceptionHub = hasDate(o.dateReceptionHubInjection);
      const pasDeRDV = isDateEmpty(o.datePremierRDVCCV);
      return aReceptionHub && pasDeRDV;
    });
  };

  // RDV PRIS : A un RDV MAIS pas de livraison
  const getCommandesRDVPris = (ordersList) => {
    return ordersList.filter(o => {
      const aUnRDV = hasDate(o.datePremierRDVCCV);
      const pasDeLivraison = isDateEmpty(o.dateLivraisonClient);
      return aUnRDV && pasDeLivraison;
    });
  };

  // RETARDS : RDV < Dernier événement ET pas de livraison
  const getCommandesRetards = (ordersList) => {
    return ordersList.filter(o => {
      const aUnRDV = hasDate(o.datePremierRDVCCV);
      const aUnDernierEvent = hasDate(o.dateDernierEvenementCCV);
      const pasDeLivraison = isDateEmpty(o.dateLivraisonClient);
      
      if (!aUnRDV || !aUnDernierEvent || !pasDeLivraison) return false;
      
      try {
        const rdvDate = new Date(o.datePremierRDVCCV);
        const lastEventDate = new Date(o.dateDernierEvenementCCV);
        rdvDate.setHours(0, 0, 0, 0);
        lastEventDate.setHours(0, 0, 0, 0);
        return rdvDate < lastEventDate;
      } catch {
        return false;
      }
    });
  };

  // Calcul des KPIs pour un hub donné
  const calculateHubStats = (hubOrders) => {
    if (hubOrders.length === 0) return null;

    const totalCommandes = hubOrders.length;
    
    const commandesLivrees = hubOrders.filter(o => o.dateLivraisonClient && o.dateLivraisonClient !== '-').length;
    const tauxLivre = totalCommandes > 0 ? ((commandesLivrees / totalCommandes) * 100).toFixed(1) : '0.0';

    const delaisMMKHub = hubOrders
      .filter(o => o.dateChargementMMK && o.dateReceptionHubInjection)
      .map(o => calculateDays(o.dateChargementMMK, o.dateReceptionHubInjection))
      .filter(d => d !== null && d >= 0);
    
    const delaiMoyenMMKHub = delaisMMKHub.length > 0 
      ? (delaisMMKHub.reduce((a, b) => a + b, 0) / delaisMMKHub.length).toFixed(1)
      : 'N/A';

    const delaisMMKRDV = hubOrders
      .filter(o => o.dateChargementMMK && o.datePremierRDVCCV)
      .map(o => calculateDays(o.dateChargementMMK, o.datePremierRDVCCV))
      .filter(d => d !== null && d >= 0);
    
    const delaiMoyenMMKRDV = delaisMMKRDV.length > 0
      ? (delaisMMKRDV.reduce((a, b) => a + b, 0) / delaisMMKRDV.length).toFixed(1)
      : 'N/A';

    const delaisMMKLivre = hubOrders
      .filter(o => o.dateChargementMMK && o.dateLivraisonClient)
      .map(o => calculateDays(o.dateChargementMMK, o.dateLivraisonClient))
      .filter(d => d !== null && d >= 0);
    
    const delaiMoyenMMKLivre = delaisMMKLivre.length > 0
      ? (delaisMMKLivre.reduce((a, b) => a + b, 0) / delaisMMKLivre.length).toFixed(1)
      : 'N/A';

    const attenteHub = hubOrders.filter(o => {
      const hasReceptionHub = o.dateReceptionHubInjection && o.dateReceptionHubInjection !== '-';
      const notDelivered = !o.dateLivraisonClient || o.dateLivraisonClient === '-';
      return hasReceptionHub && notDelivered;
    }).length;

    const sansRDV = hubOrders.filter(o => {
      const noRDV = !o.datePremierRDVCCV || o.datePremierRDVCCV === '-';
      const notDelivered = !o.dateLivraisonClient || o.dateLivraisonClient === '-';
      return noRDV && notDelivered;
    }).length;

    const today = new Date();
    today.setHours(0, 0, 0, 0);
    
    const retards = hubOrders.filter(o => {
      if (!o.datePremierRDVCCV || o.datePremierRDVCCV === '-') return false;
      if (o.dateLivraisonClient && o.dateLivraisonClient !== '-') return false;
      
      try {
        const rdvDate = new Date(o.datePremierRDVCCV);
        rdvDate.setHours(0, 0, 0, 0);
        return rdvDate < today;
      } catch {
        return false;
      }
    }).length;

    return {
      totalCommandes,
      commandesLivrees,
      tauxLivre,
      delaiMoyenMMKHub,
      delaiMoyenMMKRDV,
      delaiMoyenMMKLivre,
      attenteHub,
      sansRDV,
      retards,
    };
  };

  // Extraction des hubs uniques pour le filtre
  const uniqueHubs = useMemo(() => {
    const hubs = [...new Set(orders.map(o => o.codeHubInjection).filter(h => h))];
    return hubs.sort();
  }, [orders]);

  // Stats calculées avec useMemo (se recalculent quand filteredOrders change)
  const statsTotal = useMemo(() => {
    if (filteredOrders.length === 0) return null;
    
    // Calculer directement ici au lieu d'utiliser calculateHubStats
    const totalCommandes = filteredOrders.length;
    const commandesLivrees = filteredOrders.filter(o => o.dateLivraisonClient && o.dateLivraisonClient !== '-').length;
    const tauxLivre = totalCommandes > 0 ? ((commandesLivrees / totalCommandes) * 100).toFixed(1) : '0.0';

    // Délais moyens
    const delaisMMKHub = filteredOrders
      .filter(o => o.dateChargementMMK && o.dateReceptionHubInjection)
      .map(o => calculateDays(o.dateChargementMMK, o.dateReceptionHubInjection))
      .filter(d => d !== null && d >= 0);
    const delaiMoyenMMKHub = delaisMMKHub.length > 0 
      ? (delaisMMKHub.reduce((a, b) => a + b, 0) / delaisMMKHub.length).toFixed(1) : 'N/A';

    const delaisMMKRDV = filteredOrders
      .filter(o => o.dateChargementMMK && o.datePremierRDVCCV)
      .map(o => calculateDays(o.dateChargementMMK, o.datePremierRDVCCV))
      .filter(d => d !== null && d >= 0);
    const delaiMoyenMMKRDV = delaisMMKRDV.length > 0
      ? (delaisMMKRDV.reduce((a, b) => a + b, 0) / delaisMMKRDV.length).toFixed(1) : 'N/A';

    const delaisMMKLivre = filteredOrders
      .filter(o => o.dateChargementMMK && o.dateLivraisonClient)
      .map(o => calculateDays(o.dateChargementMMK, o.dateLivraisonClient))
      .filter(d => d !== null && d >= 0);
    const delaiMoyenMMKLivre = delaisMMKLivre.length > 0
      ? (delaisMMKLivre.reduce((a, b) => a + b, 0) / delaisMMKLivre.length).toFixed(1) : 'N/A';

    // Utiliser directement les fonctions d'export pour les compteurs
    const attenteHub = getCommandesAttenteHub(filteredOrders).length;
    const sansRDV = getCommandesSansRDV(filteredOrders).length;
    const rdvPris = getCommandesRDVPris(filteredOrders).length;
    const retards = getCommandesRetards(filteredOrders).length;

    return {
      totalCommandes,
      commandesLivrees,
      tauxLivre,
      delaiMoyenMMKHub,
      delaiMoyenMMKRDV,
      delaiMoyenMMKLivre,
      attenteHub,
      sansRDV,
      rdvPris,
      retards,
    };
  }, [filteredOrders]);

  const statsANZ = useMemo(() => {
    if (filteredOrders.length === 0) return null;
    const ordersANZ = filteredOrders.filter(o => o.codeHubInjection === 'ANZ');
    
    const totalCommandes = ordersANZ.length;
    const commandesLivrees = ordersANZ.filter(o => o.dateLivraisonClient && o.dateLivraisonClient !== '-').length;
    const tauxLivre = totalCommandes > 0 ? ((commandesLivrees / totalCommandes) * 100).toFixed(1) : '0.0';

    const delaisMMKHub = ordersANZ
      .filter(o => o.dateChargementMMK && o.dateReceptionHubInjection)
      .map(o => calculateDays(o.dateChargementMMK, o.dateReceptionHubInjection))
      .filter(d => d !== null && d >= 0);
    const delaiMoyenMMKHub = delaisMMKHub.length > 0 
      ? (delaisMMKHub.reduce((a, b) => a + b, 0) / delaisMMKHub.length).toFixed(1) : 'N/A';

    const delaisMMKRDV = ordersANZ
      .filter(o => o.dateChargementMMK && o.datePremierRDVCCV)
      .map(o => calculateDays(o.dateChargementMMK, o.datePremierRDVCCV))
      .filter(d => d !== null && d >= 0);
    const delaiMoyenMMKRDV = delaisMMKRDV.length > 0
      ? (delaisMMKRDV.reduce((a, b) => a + b, 0) / delaisMMKRDV.length).toFixed(1) : 'N/A';

    const delaisMMKLivre = ordersANZ
      .filter(o => o.dateChargementMMK && o.dateLivraisonClient)
      .map(o => calculateDays(o.dateChargementMMK, o.dateLivraisonClient))
      .filter(d => d !== null && d >= 0);
    const delaiMoyenMMKLivre = delaisMMKLivre.length > 0
      ? (delaisMMKLivre.reduce((a, b) => a + b, 0) / delaisMMKLivre.length).toFixed(1) : 'N/A';

    return {
      totalCommandes,
      commandesLivrees,
      tauxLivre,
      delaiMoyenMMKHub,
      delaiMoyenMMKRDV,
      delaiMoyenMMKLivre,
      attenteHub: getCommandesAttenteHub(ordersANZ).length,
      sansRDV: getCommandesSansRDV(ordersANZ).length,
      rdvPris: getCommandesRDVPris(ordersANZ).length,
      retards: getCommandesRetards(ordersANZ).length,
    };
  }, [filteredOrders]);

  const statsSMD = useMemo(() => {
    if (filteredOrders.length === 0) return null;
    const ordersSMD = filteredOrders.filter(o => o.codeHubInjection === 'SMD');
    
    const totalCommandes = ordersSMD.length;
    const commandesLivrees = ordersSMD.filter(o => o.dateLivraisonClient && o.dateLivraisonClient !== '-').length;
    const tauxLivre = totalCommandes > 0 ? ((commandesLivrees / totalCommandes) * 100).toFixed(1) : '0.0';

    const delaisMMKHub = ordersSMD
      .filter(o => o.dateChargementMMK && o.dateReceptionHubInjection)
      .map(o => calculateDays(o.dateChargementMMK, o.dateReceptionHubInjection))
      .filter(d => d !== null && d >= 0);
    const delaiMoyenMMKHub = delaisMMKHub.length > 0 
      ? (delaisMMKHub.reduce((a, b) => a + b, 0) / delaisMMKHub.length).toFixed(1) : 'N/A';

    const delaisMMKRDV = ordersSMD
      .filter(o => o.dateChargementMMK && o.datePremierRDVCCV)
      .map(o => calculateDays(o.dateChargementMMK, o.datePremierRDVCCV))
      .filter(d => d !== null && d >= 0);
    const delaiMoyenMMKRDV = delaisMMKRDV.length > 0
      ? (delaisMMKRDV.reduce((a, b) => a + b, 0) / delaisMMKRDV.length).toFixed(1) : 'N/A';

    const delaisMMKLivre = ordersSMD
      .filter(o => o.dateChargementMMK && o.dateLivraisonClient)
      .map(o => calculateDays(o.dateChargementMMK, o.dateLivraisonClient))
      .filter(d => d !== null && d >= 0);
    const delaiMoyenMMKLivre = delaisMMKLivre.length > 0
      ? (delaisMMKLivre.reduce((a, b) => a + b, 0) / delaisMMKLivre.length).toFixed(1) : 'N/A';

    return {
      totalCommandes,
      commandesLivrees,
      tauxLivre,
      delaiMoyenMMKHub,
      delaiMoyenMMKRDV,
      delaiMoyenMMKLivre,
      attenteHub: getCommandesAttenteHub(ordersSMD).length,
      sansRDV: getCommandesSansRDV(ordersSMD).length,
      rdvPris: getCommandesRDVPris(ordersSMD).length,
      retards: getCommandesRetards(ordersSMD).length,
    };
  }, [filteredOrders]);

  // Template Excel à télécharger
  const downloadTemplate = () => {
    const template = [
      {
        'Commande': 'CMD-001',
        'Date': '01/12/2024',
        'Client': 'Client A',
        'CP Destination': '75001',
        'Ville Destination': 'Paris',
        'date Chargement mmk': '02/12/2024',
        'Code': 'CODE001',
        'CODE HUB INJECTION': 'ANZ',
        'DATE CREATION ANNONCE CCV': '02/12/2024',
        'DATE LIVRAISON CLIENT': '07/12/2024',
        'DATE DERNIER EVENEMENT CCV': '07/12/2024',
        'DERNIER EVENEMENT CCV': 'Livré',
        'DATE DU PREMIER RDV CCV': '06/12/2024',
        'DATE RECEPTION HUB INJECTION': '04/12/2024',
      },
      {
        'Commande': 'CMD-002',
        'Date': '03/12/2024',
        'Client': 'Client B',
        'CP Destination': '69000',
        'Ville Destination': 'Lyon',
        'date Chargement mmk': '04/12/2024',
        'Code': 'CODE002',
        'CODE HUB INJECTION': 'SMD',
        'DATE CREATION ANNONCE CCV': '04/12/2024',
        'DATE LIVRAISON CLIENT': '',
        'DATE DERNIER EVENEMENT CCV': '08/12/2024',
        'DERNIER EVENEMENT CCV': 'En transit',
        'DATE DU PREMIER RDV CCV': '08/12/2024',
        'DATE RECEPTION HUB INJECTION': '05/12/2024',
      },
    ];

    const ws = XLSX.utils.json_to_sheet(template);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, 'Commandes');
    XLSX.writeFile(wb, 'template_commandes_CCV.xlsx');
  };

  // Export des données filtrées
  const exportToExcel = () => {
    if (filteredOrders.length === 0) {
      alert('Aucune donnée à exporter');
      return;
    }

    const exportData = filteredOrders.map(o => ({
      'Commande': o.commande,
      'Date': formatDateDisplay(o.dateCommande),
      'Client': o.client,
      'CP Destination': o.cpDestination,
      'Ville Destination': o.villeDestination,
      'Code': o.code,
      'date Chargement mmk': formatDateDisplay(o.dateChargementMMK),
      'CODE HUB INJECTION': o.codeHubInjection,
      'DATE CREATION ANNONCE CCV': formatDateDisplay(o.dateCreationAnnonceCCV),
      'DATE LIVRAISON CLIENT': formatDateDisplay(o.dateLivraisonClient),
      'DATE DERNIER EVENEMENT CCV': formatDateDisplay(o.dateDernierEvenementCCV),
      'DERNIER EVENEMENT CCV': o.dernierEvenementCCV,
      'DATE DU PREMIER RDV CCV': formatDateDisplay(o.datePremierRDVCCV),
      'DATE RECEPTION HUB INJECTION': formatDateDisplay(o.dateReceptionHubInjection),
    }));

    const ws = XLSX.utils.json_to_sheet(exportData);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, 'Export');
    XLSX.writeFile(wb, `export_commandes_${new Date().toISOString().split('T')[0]}.xlsx`);
  };

  // Export Attente HUB
  const exportAttenteHub = () => {
    const commandesAttenteHub = getCommandesAttenteHub(filteredOrders);

    if (commandesAttenteHub.length === 0) {
      alert('Aucune commande en attente HUB');
      return;
    }

    const exportData = commandesAttenteHub.map(o => ({
      'Commande': o.commande,
      'Date': formatDateDisplay(o.dateCommande),
      'Client': o.client,
      'CP Destination': o.cpDestination,
      'Ville Destination': o.villeDestination,
      'Code': o.code,
      'CODE HUB INJECTION': o.codeHubInjection,
      'date Chargement mmk': formatDateDisplay(o.dateChargementMMK),
      'DATE CREATION ANNONCE CCV': formatDateDisplay(o.dateCreationAnnonceCCV),
      'DERNIER EVENEMENT CCV': o.dernierEvenementCCV,
      'DATE DERNIER EVENEMENT CCV': formatDateDisplay(o.dateDernierEvenementCCV),
    }));

    const ws = XLSX.utils.json_to_sheet(exportData);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, 'Attente HUB');
    XLSX.writeFile(wb, `attente_hub_${new Date().toISOString().split('T')[0]}.xlsx`);
  };

  // DEBUG : Export ce qui est réellement compté dans statsTotal.attenteHub
  const exportAttenteHubDebug = () => {
    // Recalculer EXACTEMENT comme dans calculateHubStats
    const hubOrders = filteredOrders; // Le même paramètre passé pour statsTotal
    const commandesDebug = hubOrders.filter(o => {
      const pasDeReceptionHub = isDateEmpty(o.dateReceptionHubInjection);
      const pasDeRDV = isDateEmpty(o.datePremierRDVCCV);
      const pasDeLivraison = isDateEmpty(o.dateLivraisonClient);
      return pasDeReceptionHub && pasDeRDV && pasDeLivraison;
    });

    alert(`DEBUG: Trouvé ${commandesDebug.length} commandes (affiché: ${statsTotal?.attenteHub})`);

    if (commandesDebug.length === 0) {
      alert('Aucune commande trouvée en debug');
      return;
    }

    const exportData = commandesDebug.map(o => ({
      'Commande': o.commande,
      'Client': o.client,
      'Hub': o.codeHubInjection,
      'dateReceptionHubInjection': o.dateReceptionHubInjection || 'VIDE',
      'datePremierRDVCCV': o.datePremierRDVCCV || 'VIDE',
      'dateLivraisonClient': o.dateLivraisonClient || 'VIDE',
      'DERNIER EVENEMENT': o.dernierEvenementCCV,
    }));

    const ws = XLSX.utils.json_to_sheet(exportData);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, 'Debug Attente HUB');
    XLSX.writeFile(wb, `DEBUG_attente_hub_${new Date().toISOString().split('T')[0]}.xlsx`);
  };

  // Export des commandes sans RDV (utilise la même fonction pour garantir cohérence)
  const exportSansRDV = () => {
    const commandesSansRDV = getCommandesSansRDV(filteredOrders);

    if (commandesSansRDV.length === 0) {
      alert('Aucune commande sans RDV');
      return;
    }

    const exportData = commandesSansRDV.map(o => ({
      'Commande': o.commande,
      'Date': formatDateDisplay(o.dateCommande),
      'Client': o.client,
      'CP Destination': o.cpDestination,
      'Ville Destination': o.villeDestination,
      'Code': o.code,
      'CODE HUB INJECTION': o.codeHubInjection,
      'DATE RECEPTION HUB INJECTION': formatDateDisplay(o.dateReceptionHubInjection),
      'date Chargement mmk': formatDateDisplay(o.dateChargementMMK),
      'DERNIER EVENEMENT CCV': o.dernierEvenementCCV,
    }));

    const ws = XLSX.utils.json_to_sheet(exportData);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, 'Sans RDV');
    XLSX.writeFile(wb, `sans_rdv_${new Date().toISOString().split('T')[0]}.xlsx`);
  };

  // Export des commandes avec RDV pris (utilise la même fonction pour garantir cohérence)
  const exportRDVPris = () => {
    const commandesRDVPris = getCommandesRDVPris(filteredOrders);

    if (commandesRDVPris.length === 0) {
      alert('Aucune commande avec RDV pris');
      return;
    }

    const exportData = commandesRDVPris.map(o => ({
      'Commande': o.commande,
      'Date': formatDateDisplay(o.dateCommande),
      'Client': o.client,
      'CP Destination': o.cpDestination,
      'Ville Destination': o.villeDestination,
      'Code': o.code,
      'CODE HUB INJECTION': o.codeHubInjection,
      'DATE DU PREMIER RDV CCV': formatDateDisplay(o.datePremierRDVCCV),
      'DATE RECEPTION HUB INJECTION': formatDateDisplay(o.dateReceptionHubInjection),
      'DERNIER EVENEMENT CCV': o.dernierEvenementCCV,
    }));

    const ws = XLSX.utils.json_to_sheet(exportData);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, 'RDV Pris');
    XLSX.writeFile(wb, `rdv_pris_${new Date().toISOString().split('T')[0]}.xlsx`);
  };

  // Export des commandes en retard (utilise la même fonction pour garantir cohérence)
  const exportRetards = () => {
    const commandesRetards = getCommandesRetards(filteredOrders);

    if (commandesRetards.length === 0) {
      alert('Aucune commande en retard');
      return;
    }

    const exportData = commandesRetards.map(o => ({
      'Commande': o.commande,
      'Date': formatDateDisplay(o.dateCommande),
      'Client': o.client,
      'CP Destination': o.cpDestination,
      'Ville Destination': o.villeDestination,
      'Code': o.code,
      'CODE HUB INJECTION': o.codeHubInjection,
      'DATE DU PREMIER RDV CCV': formatDateDisplay(o.datePremierRDVCCV),
      'DATE DERNIER EVENEMENT CCV': formatDateDisplay(o.dateDernierEvenementCCV),
      'DERNIER EVENEMENT CCV': o.dernierEvenementCCV,
    }));

    const ws = XLSX.utils.json_to_sheet(exportData);
    const wb = XLSX.utils.book_new();
    XLSX.utils.book_append_sheet(wb, ws, 'Retards');
    XLSX.writeFile(wb, `retards_${new Date().toISOString().split('T')[0]}.xlsx`);
  };

  // Export PDF des statistiques
  const exportStatsPDF = () => {
    if (!statsTotal) {
      alert('Aucune donnée à exporter');
      return;
    }

    const content = `
      <!DOCTYPE html>
      <html>
      <head>
        <meta charset="UTF-8">
        <style>
          body { font-family: Arial, sans-serif; padding: 40px; }
          h1 { color: #1e293b; border-bottom: 3px solid #3b82f6; padding-bottom: 10px; }
          .date { color: #64748b; font-size: 14px; margin-bottom: 30px; }
          .hub-section { margin-bottom: 40px; padding: 20px; border: 2px solid #e2e8f0; border-radius: 8px; }
          .hub-title { font-size: 24px; font-weight: bold; margin-bottom: 20px; padding-bottom: 10px; border-bottom: 2px solid; }
          .total-title { color: #475569; border-color: #94a3b8; }
          .anz-title { color: #3b82f6; border-color: #93c5fd; }
          .smd-title { color: #8b5cf6; border-color: #c4b5fd; }
          .stats-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 15px; }
          .stat-item { padding: 15px; background: #f8fafc; border-radius: 6px; }
          .stat-label { font-size: 14px; color: #64748b; margin-bottom: 5px; }
          .stat-value { font-size: 28px; font-weight: bold; color: #1e293b; }
          .alert-section { margin-top: 30px; padding: 20px; background: #fef3c7; border-left: 4px solid #f59e0b; }
          .alert-title { font-weight: bold; margin-bottom: 10px; }
        </style>
      </head>
      <body>
        <h1>📊 Rapport Dashboard Livraisons CCV</h1>
        <div class="date">Généré le ${new Date().toLocaleDateString('fr-FR')} à ${new Date().toLocaleTimeString('fr-FR')}</div>
        
        <div class="hub-section">
          <div class="hub-title total-title">📊 TOTAL</div>
          <div class="stats-grid">
            <div class="stat-item"><div class="stat-label">Total commandes</div><div class="stat-value">${statsTotal.totalCommandes}</div></div>
            <div class="stat-item"><div class="stat-label">Taux livré</div><div class="stat-value">${statsTotal.tauxLivre}%</div></div>
            <div class="stat-item"><div class="stat-label">Délai : Chargement MMK → HUB</div><div class="stat-value">${statsTotal.delaiMoyenMMKHub}j</div></div>
            <div class="stat-item"><div class="stat-label">Délai : Chargement MMK → 1er RDV</div><div class="stat-value">${statsTotal.delaiMoyenMMKRDV}j</div></div>
            <div class="stat-item"><div class="stat-label">Délai : Chargement MMK → Livré</div><div class="stat-value">${statsTotal.delaiMoyenMMKLivre}j</div></div>
            <div class="stat-item"><div class="stat-label">Attente HUB</div><div class="stat-value">${statsTotal.attenteHub}</div></div>
            <div class="stat-item"><div class="stat-label">Sans RDV</div><div class="stat-value">${statsTotal.sansRDV}</div></div>
            <div class="stat-item"><div class="stat-label">RDV Pris</div><div class="stat-value">${statsTotal.rdvPris}</div></div>
            <div class="stat-item"><div class="stat-label">Retards</div><div class="stat-value">${statsTotal.retards}</div></div>
          </div>
        </div>

        ${statsANZ ? `
        <div class="hub-section">
          <div class="hub-title anz-title">🏢 HUB ANZ</div>
          <div class="stats-grid">
            <div class="stat-item"><div class="stat-label">Total commandes</div><div class="stat-value">${statsANZ.totalCommandes}</div></div>
            <div class="stat-item"><div class="stat-label">Taux livré</div><div class="stat-value">${statsANZ.tauxLivre}%</div></div>
            <div class="stat-item"><div class="stat-label">Délai : Chargement MMK → HUB</div><div class="stat-value">${statsANZ.delaiMoyenMMKHub}j</div></div>
            <div class="stat-item"><div class="stat-label">Délai : Chargement MMK → 1er RDV</div><div class="stat-value">${statsANZ.delaiMoyenMMKRDV}j</div></div>
            <div class="stat-item"><div class="stat-label">Délai : Chargement MMK → Livré</div><div class="stat-value">${statsANZ.delaiMoyenMMKLivre}j</div></div>
            <div class="stat-item"><div class="stat-label">Attente HUB</div><div class="stat-value">${statsANZ.attenteHub}</div></div>
            <div class="stat-item"><div class="stat-label">Sans RDV</div><div class="stat-value">${statsANZ.sansRDV}</div></div>
            <div class="stat-item"><div class="stat-label">RDV Pris</div><div class="stat-value">${statsANZ.rdvPris}</div></div>
            <div class="stat-item"><div class="stat-label">Retards</div><div class="stat-value">${statsANZ.retards}</div></div>
          </div>
        </div>
        ` : ''}

        ${statsSMD ? `
        <div class="hub-section">
          <div class="hub-title smd-title">🏢 HUB SMD</div>
          <div class="stats-grid">
            <div class="stat-item"><div class="stat-label">Total commandes</div><div class="stat-value">${statsSMD.totalCommandes}</div></div>
            <div class="stat-item"><div class="stat-label">Taux livré</div><div class="stat-value">${statsSMD.tauxLivre}%</div></div>
            <div class="stat-item"><div class="stat-label">Délai : Chargement MMK → HUB</div><div class="stat-value">${statsSMD.delaiMoyenMMKHub}j</div></div>
            <div class="stat-item"><div class="stat-label">Délai : Chargement MMK → 1er RDV</div><div class="stat-value">${statsSMD.delaiMoyenMMKRDV}j</div></div>
            <div class="stat-item"><div class="stat-label">Délai : Chargement MMK → Livré</div><div class="stat-value">${statsSMD.delaiMoyenMMKLivre}j</div></div>
            <div class="stat-item"><div class="stat-label">Attente HUB</div><div class="stat-value">${statsSMD.attenteHub}</div></div>
            <div class="stat-item"><div class="stat-label">Sans RDV</div><div class="stat-value">${statsSMD.sansRDV}</div></div>
            <div class="stat-item"><div class="stat-label">RDV Pris</div><div class="stat-value">${statsSMD.rdvPris}</div></div>
            <div class="stat-item"><div class="stat-label">Retards</div><div class="stat-value">${statsSMD.retards}</div></div>
          </div>
        </div>
        ` : ''}

        ${(statsTotal.attenteHub > 0 || statsTotal.sansRDV > 0 || statsTotal.rdvPris > 0 || statsTotal.retards > 0) ? `
        <div class="alert-section">
          <div class="alert-title">⚠️ Points d'attention</div>
          ${statsTotal.attenteHub > 0 ? `<p>• ${statsTotal.attenteHub} commandes en attente HUB (sans réception)</p>` : ''}
          ${statsTotal.sansRDV > 0 ? `<p>• ${statsTotal.sansRDV} commandes sans RDV planifié</p>` : ''}
          ${statsTotal.rdvPris > 0 ? `<p>• ${statsTotal.rdvPris} commandes avec RDV pris (en cours)</p>` : ''}
          ${statsTotal.retards > 0 ? `<p>• ${statsTotal.retards} commandes en retard</p>` : ''}
        </div>
        ` : ''}
      </body>
      </html>
    `;

    const blob = new Blob([content], { type: 'text/html' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `rapport_stats_${new Date().toISOString().split('T')[0]}.html`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
    
    alert('Rapport téléchargé ! Ouvrez le fichier HTML et utilisez la fonction "Imprimer > Enregistrer au format PDF" de votre navigateur.');
  };

  // Rafraîchir les données
  const handleRefresh = () => {
    document.getElementById('file-upload').click();
  };

  // Reset des filtres
  const resetFilters = () => {
    setTempSelectedHub('TOUS');
    setTempDateCommandeStart('');
    setTempDateCommandeEnd('');
    setTempDateChargementStart('');
    setTempDateChargementEnd('');
    setSelectedHub('TOUS');
    setDateCommandeStart('');
    setDateCommandeEnd('');
    setDateChargementStart('');
    setDateChargementEnd('');
  };

  // Appliquer les filtres
  const applyFilters = () => {
    setSelectedHub(tempSelectedHub);
    setDateCommandeStart(tempDateCommandeStart);
    setDateCommandeEnd(tempDateCommandeEnd);
    setDateChargementStart(tempDateChargementStart);
    setDateChargementEnd(tempDateChargementEnd);
  };

  const KPICard = ({ title, stats, color = 'blue' }) => (
    <div className="bg-white rounded-xl shadow-md p-5">
      <h3 className={`text-lg font-bold text-${color}-700 mb-4 border-b pb-2`}>{title}</h3>
      <div className="space-y-3">
        <div className="flex justify-between items-center">
          <span className="text-sm text-slate-600">Total commandes</span>
          <span className="text-xl font-bold text-slate-800">{stats?.totalCommandes || 0}</span>
        </div>
        <div className="flex justify-between items-center">
          <span className="text-sm text-slate-600">Taux livré</span>
          <span className="text-xl font-bold text-green-600">{stats?.tauxLivre || 0}%</span>
        </div>
        <div className="flex justify-between items-center">
          <span className="text-sm text-slate-600">Délai : Chargement MMK → HUB</span>
          <span className="text-lg font-semibold text-slate-700">{stats?.delaiMoyenMMKHub || 'N/A'}j</span>
        </div>
        <div className="flex justify-between items-center">
          <span className="text-sm text-slate-600">Délai : Chargement MMK → 1er RDV</span>
          <span className="text-lg font-semibold text-slate-700">{stats?.delaiMoyenMMKRDV || 'N/A'}j</span>
        </div>
        <div className="flex justify-between items-center">
          <span className="text-sm text-slate-600">Délai : Chargement MMK → Livré</span>
          <span className="text-lg font-semibold text-slate-700">{stats?.delaiMoyenMMKLivre || 'N/A'}j</span>
        </div>
        <div className="flex justify-between items-center pt-2 border-t">
          <span className="text-sm text-slate-600">Attente HUB</span>
          <span className="text-lg font-semibold text-orange-600">{stats?.attenteHub || 0}</span>
        </div>
        <div className="flex justify-between items-center">
          <span className="text-sm text-slate-600">Sans RDV</span>
          <span className="text-lg font-semibold text-blue-600">{stats?.sansRDV || 0}</span>
        </div>
        <div className="flex justify-between items-center">
          <span className="text-sm text-slate-600">RDV Pris</span>
          <span className="text-lg font-semibold text-green-600">{stats?.rdvPris || 0}</span>
        </div>
        <div className="flex justify-between items-center">
          <span className="text-sm text-slate-600">Retards</span>
          <span className="text-lg font-semibold text-red-600">{stats?.retards || 0}</span>
        </div>
      </div>
    </div>
  );

  return (
    <div className="min-h-screen bg-gradient-to-br from-slate-50 to-slate-100 p-6">
      <div className="max-w-7xl mx-auto">
        <div className="mb-8">
          <h1 className="text-4xl font-bold text-slate-800 mb-2">Dashboard Livraisons CCV</h1>
          <p className="text-slate-600">Suivi détaillé par HUB d'injection</p>
        </div>

        <div className="bg-white rounded-xl shadow-lg p-6 mb-6">
          <div className="flex items-center justify-between mb-4 flex-wrap gap-3">
            <div className="flex items-center gap-3">
              <Upload className="text-blue-600" size={24} />
              <h2 className="text-xl font-semibold text-slate-800">Import & Actions</h2>
            </div>
            <div className="flex gap-2 flex-wrap">
              <button
                onClick={downloadTemplate}
                className="flex items-center gap-2 px-4 py-2 bg-slate-100 hover:bg-slate-200 text-slate-700 rounded-lg transition"
              >
                <Download size={18} />
                Template
              </button>
              <button
                onClick={handleRefresh}
                className="flex items-center gap-2 px-4 py-2 bg-blue-100 hover:bg-blue-200 text-blue-700 rounded-lg transition"
              >
                <RefreshCw size={18} />
                Rafraîchir
              </button>
              <button
                onClick={() => setShowFilters(!showFilters)}
                className="flex items-center gap-2 px-4 py-2 bg-purple-100 hover:bg-purple-200 text-purple-700 rounded-lg transition"
              >
                <Filter size={18} />
                Filtres
              </button>
              <button
                onClick={exportToExcel}
                disabled={filteredOrders.length === 0}
                className="flex items-center gap-2 px-4 py-2 bg-green-100 hover:bg-green-200 text-green-700 rounded-lg transition disabled:opacity-50 disabled:cursor-not-allowed"
              >
                <FileDown size={18} />
                Exporter
              </button>
              <button
                onClick={exportStatsPDF}
                disabled={!statsTotal}
                className="flex items-center gap-2 px-4 py-2 bg-red-100 hover:bg-red-200 text-red-700 rounded-lg transition disabled:opacity-50 disabled:cursor-not-allowed"
              >
                <FileDown size={18} />
                PDF Stats
              </button>
              <button
                onClick={exportAttenteHubDebug}
                disabled={!statsTotal}
                className="flex items-center gap-2 px-4 py-2 bg-yellow-100 hover:bg-yellow-200 text-yellow-700 rounded-lg transition disabled:opacity-50 disabled:cursor-not-allowed"
                title="Debug: Exporter les 593 commandes comptées"
              >
                🐛 Debug 593
              </button>
            </div>
          </div>
          
          <div className="border-2 border-dashed border-slate-300 rounded-lg p-8 text-center hover:border-blue-400 transition">
            <input
              type="file"
              accept=".xlsx,.xls"
              onChange={handleFileUpload}
              className="hidden"
              id="file-upload"
            />
            <label htmlFor="file-upload" className="cursor-pointer">
              <div className="flex flex-col items-center gap-3">
                <Package size={48} className="text-slate-400" />
                <p className="text-lg font-medium text-slate-700">
                  Cliquez pour importer votre fichier Excel CCV
                </p>
                <p className="text-sm text-slate-500">Formats acceptés: .xlsx, .xls</p>
              </div>
            </label>
          </div>

          {loading && (
            <div className="mt-4 text-center text-blue-600 font-medium">
              Chargement en cours...
            </div>
          )}
        </div>

        {showFilters && orders.length > 0 && (
          <div className="bg-white rounded-xl shadow-lg p-6 mb-6">
            <div className="flex items-center justify-between mb-4">
              <h3 className="text-lg font-semibold text-slate-800">Filtres</h3>
              <button
                onClick={resetFilters}
                className="text-sm text-blue-600 hover:text-blue-800 font-medium"
              >
                Réinitialiser
              </button>
            </div>
            <div className="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-5 gap-4 mb-4">
              <div>
                <label className="block text-sm font-medium text-slate-700 mb-2">Hub Injection</label>
                <select
                  value={tempSelectedHub}
                  onChange={(e) => setTempSelectedHub(e.target.value)}
                  className="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                >
                  <option value="TOUS">Tous</option>
                  {uniqueHubs.map(hub => (
                    <option key={hub} value={hub}>{hub}</option>
                  ))}
                </select>
              </div>
              <div>
                <label className="block text-sm font-medium text-slate-700 mb-2">Date commande (début)</label>
                <input
                  type="date"
                  value={tempDateCommandeStart}
                  onChange={(e) => setTempDateCommandeStart(e.target.value)}
                  className="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                />
              </div>
              <div>
                <label className="block text-sm font-medium text-slate-700 mb-2">Date commande (fin)</label>
                <input
                  type="date"
                  value={tempDateCommandeEnd}
                  onChange={(e) => setTempDateCommandeEnd(e.target.value)}
                  className="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                />
              </div>
              <div>
                <label className="block text-sm font-medium text-slate-700 mb-2">Date chargement (début)</label>
                <input
                  type="date"
                  value={tempDateChargementStart}
                  onChange={(e) => setTempDateChargementStart(e.target.value)}
                  className="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                />
              </div>
              <div>
                <label className="block text-sm font-medium text-slate-700 mb-2">Date chargement (fin)</label>
                <input
                  type="date"
                  value={tempDateChargementEnd}
                  onChange={(e) => setTempDateChargementEnd(e.target.value)}
                  className="w-full px-3 py-2 border border-slate-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                />
              </div>
            </div>
            <div className="flex justify-end">
              <button
                onClick={applyFilters}
                className="flex items-center gap-2 px-6 py-2 bg-blue-600 hover:bg-blue-700 text-white rounded-lg transition font-medium shadow-md"
              >
                <Filter size={18} />
                Appliquer les filtres
              </button>
            </div>
          </div>
        )}

        {statsTotal && (
          <>
            <div className="grid grid-cols-1 lg:grid-cols-3 gap-6 mb-6">
              <KPICard title="📊 TOTAL" stats={statsTotal} color="slate" />
              <KPICard title="🏢 HUB ANZ" stats={statsANZ} color="blue" />
              <KPICard title="🏢 HUB SMD" stats={statsSMD} color="purple" />
            </div>

            <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-6">
              <div className="bg-gradient-to-r from-orange-50 to-orange-100 border-l-4 border-orange-500 rounded-lg p-4 shadow">
                <div className="flex items-center justify-between">
                  <div className="flex items-center gap-3">
                    <Clock className="text-orange-600" size={32} />
                    <div>
                      <div className="text-sm font-medium text-orange-800">Attente HUB</div>
                      <div className="text-2xl font-bold text-orange-900">{statsTotal.attenteHub}</div>
                    </div>
                  </div>
                  <button
                    onClick={exportAttenteHub}
                    disabled={statsTotal.attenteHub === 0}
                    className="p-2 bg-orange-200 hover:bg-orange-300 rounded-lg transition disabled:opacity-50 disabled:cursor-not-allowed"
                    title="Télécharger les commandes en attente HUB"
                  >
                    <Download size={18} className="text-orange-700" />
                  </button>
                </div>
              </div>
              <div className="bg-gradient-to-r from-blue-50 to-blue-100 border-l-4 border-blue-500 rounded-lg p-4 shadow">
                <div className="flex items-center justify-between">
                  <div className="flex items-center gap-3">
                    <AlertTriangle className="text-blue-600" size={32} />
                    <div>
                      <div className="text-sm font-medium text-blue-800">Sans RDV</div>
                      <div className="text-2xl font-bold text-blue-900">{statsTotal.sansRDV}</div>
                    </div>
                  </div>
                  <button
                    onClick={exportSansRDV}
                    disabled={statsTotal.sansRDV === 0}
                    className="p-2 bg-blue-200 hover:bg-blue-300 rounded-lg transition disabled:opacity-50 disabled:cursor-not-allowed"
                    title="Télécharger les commandes sans RDV"
                  >
                    <Download size={18} className="text-blue-700" />
                  </button>
                </div>
              </div>
              <div className="bg-gradient-to-r from-red-50 to-red-100 border-l-4 border-red-500 rounded-lg p-4 shadow">
                <div className="flex items-center justify-between">
                  <div className="flex items-center gap-3">
                    <AlertTriangle className="text-red-600" size={32} />
                    <div>
                      <div className="text-sm font-medium text-red-800">Retards</div>
                      <div className="text-2xl font-bold text-red-900">{statsTotal.retards}</div>
                    </div>
                  </div>
                  <button
                    onClick={exportRetards}
                    disabled={statsTotal.retards === 0}
                    className="p-2 bg-red-200 hover:bg-red-300 rounded-lg transition disabled:opacity-50 disabled:cursor-not-allowed"
                    title="Télécharger les commandes en retard"
                  >
                    <Download size={18} className="text-red-700" />
                  </button>
                </div>
              </div>
            </div>

            <div className="bg-white rounded-xl shadow-lg p-6">
              <h2 className="text-xl font-semibold text-slate-800 mb-4">
                Détail des commandes ({filteredOrders.length})
              </h2>
              <div className="overflow-x-auto">
                <table className="w-full text-sm">
                  <thead>
                    <tr className="border-b-2 border-slate-200">
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">Commande</th>
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">Date</th>
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">Client</th>
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">Destination</th>
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">Code</th>
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">Hub</th>
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">Charg. MMK</th>
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">Récep. HUB</th>
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">1er RDV</th>
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">Livraison</th>
                      <th className="text-left py-3 px-2 text-xs font-semibold text-slate-700">Dernier Événement</th>
                    </tr>
                  </thead>
                  <tbody>
                    {filteredOrders.map((order, idx) => (
                      <tr key={idx} className="border-b border-slate-100 hover:bg-slate-50 transition">
                        <td className="py-3 px-2 font-medium text-slate-800">{order.commande}</td>
                        <td className="py-3 px-2 text-slate-600">{formatDateDisplay(order.dateCommande)}</td>
                        <td className="py-3 px-2 text-slate-600">{order.client}</td>
                        <td className="py-3 px-2 text-slate-600">{order.cpDestination} {order.villeDestination}</td>
                        <td className="py-3 px-2 text-slate-600 font-mono text-xs">{order.code}</td>
                        <td className="py-3 px-2">
                          <span className={`inline-flex px-2 py-1 rounded text-xs font-semibold ${
                            order.codeHubInjection === 'ANZ' ? 'bg-blue-100 text-blue-800' : 
                            order.codeHubInjection === 'SMD' ? 'bg-purple-100 text-purple-800' :
                            'bg-gray-100 text-gray-800'
                          }`}>
                            {order.codeHubInjection}
                          </span>
                        </td>
                        <td className="py-3 px-2 text-slate-600">{formatDateDisplay(order.dateChargementMMK)}</td>
                        <td className="py-3 px-2 text-slate-600">{formatDateDisplay(order.dateReceptionHubInjection)}</td>
                        <td className="py-3 px-2 text-slate-600">{formatDateDisplay(order.datePremierRDVCCV)}</td>
                        <td className="py-3 px-2 text-slate-600">{formatDateDisplay(order.dateLivraisonClient)}</td>
                        <td className="py-3 px-2">
                          <div className="flex flex-col">
                            <span className={`inline-flex px-2 py-1 rounded-full text-xs font-medium mb-1 ${
                              order.dateLivraisonClient ? 'bg-green-100 text-green-800' :
                              order.datePremierRDVCCV && new Date(order.datePremierRDVCCV) < new Date() ? 'bg-red-100 text-red-800' :
                              'bg-yellow-100 text-yellow-800'
                            }`}>
                              {order.dernierEvenementCCV || 'En cours'}
                            </span>
                            {order.dateDernierEvenementCCV && (
                              <span className="text-xs text-slate-500">{formatDateDisplay(order.dateDernierEvenementCCV)}</span>
                            )}
                          </div>
                        </td>
                      </tr>
                    ))}
                  </tbody>
                </table>
              </div>
            </div>
          </>
        )}

        {orders.length === 0 && !loading && (
          <div className="bg-white rounded-xl shadow-lg p-12 text-center">
            <Package size={64} className="mx-auto text-slate-300 mb-4" />
            <h3 className="text-xl font-semibold text-slate-700 mb-2">
              Aucune donnée importée
            </h3>
            <p className="text-slate-500">
              Importez votre fichier Excel CCV pour commencer le suivi par HUB
            </p>
          </div>
        )}
      </div>
    </div>
  );
};

export default LogisticsDashboard;
