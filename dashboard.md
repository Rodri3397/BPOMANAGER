import React, { useState, useMemo } from 'react';
import { base44 } from '@/api/base44Client';
import { useQuery } from '@tanstack/react-query';
import { 
  TrendingUp, 
  Users, 
  Package, 
  DollarSign,
  Calendar,
  BarChart3
} from 'lucide-react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";
import {
  BarChart,
  Bar,
  XAxis,
  YAxis,
  CartesianGrid,
  Tooltip,
  ResponsiveContainer,
  Cell,
  LineChart,
  Line
} from 'recharts';

const StatCard = ({ title, value, icon: Icon, subtext, color }) => (
  <Card className="border-none shadow-sm">
    <CardHeader className="flex flex-row items-center justify-between pb-2 space-y-0">
      <CardTitle className="text-sm font-medium text-slate-500 uppercase">{title}</CardTitle>
      <div className={`p-2 rounded-full ${color} bg-opacity-10`}>
        <Icon className={`w-4 h-4 ${color.replace('bg-', 'text-')}`} />
      </div>
    </CardHeader>
    <CardContent>
      <div className="text-2xl font-bold text-slate-800">{value}</div>
      {subtext && <p className="text-xs text-slate-500 mt-1">{subtext}</p>}
    </CardContent>
  </Card>
);

export default function Dashboard() {
  const currentYear = new Date().getFullYear().toString();
  const currentMonth = (new Date().getMonth() + 1).toString();

  const [viewMode, setViewMode] = useState('month'); // month, year, all
  const [selectedYear, setSelectedYear] = useState(currentYear);
  const [selectedMonth, setSelectedMonth] = useState(currentMonth);

  const { data: items } = useQuery({
    queryKey: ['dashboard-items'],
    queryFn: () => base44.entities.PurchaseItem.list({ limit: 2000 }),
  });

  const { data: purchases } = useQuery({
    queryKey: ['dashboard-purchases'],
    queryFn: () => base44.entities.Purchase.list({ limit: 1000 }),
  });

  const stats = useMemo(() => {
    if (!items || !purchases) return null;

    const purchaseMap = new Map(purchases.map(p => [p.id, p]));
    
    // Enrich items with date
    const enrichedItems = items.map(i => {
        const p = purchaseMap.get(i.purchase_id);
        const date = p ? new Date(p.created_date) : new Date();
        return { ...i, date, purchase: p };
    });

    // Filter by Period
    const filteredItems = enrichedItems.filter(item => {
        if (viewMode === 'all') return true;
        
        const itemYear = item.date.getFullYear().toString();
        const itemMonth = (item.date.getMonth() + 1).toString();

        if (viewMode === 'year') return itemYear === selectedYear;
        if (viewMode === 'month') return itemYear === selectedYear && itemMonth === selectedMonth;
        return true;
    });

    // KPIS
    // 1. Total Purchases (Count of unique SCs in filtered items or filtered purchases?)
    // Let's count unique Purchases involved
    const uniquePurchases = new Set(filteredItems.map(i => i.purchase_id)).size;
    
    // 2. Total Spent (Sum of preco_total of items)
    const totalSpent = filteredItems.reduce((acc, i) => acc + (i.preco_total || 0), 0);
    
    // 3. Total Saving
    const totalSaving = filteredItems.reduce((acc, i) => acc + (i.saving_reais || 0), 0);

    // 4. Unique Suppliers
    const uniqueSuppliers = new Set(filteredItems.map(i => i.supplier_id).filter(Boolean)).size;

    // 5. Unique Items
    const uniqueMaterials = new Set(filteredItems.map(i => i.material_id)).size;

    // Charts Data
    
    // Buyer Volume
    const buyerStats = {};
    filteredItems.forEach(i => {
        const buyerName = i.purchase?.buyer_nome || 'N/A';
        if (!buyerStats[buyerName]) buyerStats[buyerName] = 0;
        buyerStats[buyerName] += (i.preco_total || 0);
    });
    const buyerChartData = Object.entries(buyerStats)
        .map(([name, value]) => ({ name, value }))
        .sort((a,b) => b.value - a.value)
        .slice(0, 10);

    // Supplier Volume
    const supplierStats = {};
    filteredItems.forEach(i => {
        const supName = i.supplier_nome || 'N/A';
        if (!supplierStats[supName]) supplierStats[supName] = 0;
        supplierStats[supName] += (i.preco_total || 0);
    });
    const supplierChartData = Object.entries(supplierStats)
        .map(([name, value]) => ({ name, value }))
        .sort((a,b) => b.value - a.value)
        .slice(0, 10);
        
    // Evolution (Monthly)
    const evolutionStats = {};
    // Populate all months if year view
    if (viewMode === 'year') {
        for(let m=1; m<=12; m++) evolutionStats[m] = { spent: 0, saving: 0 };
    }
    
    filteredItems.forEach(i => {
        const m = i.date.getMonth() + 1;
        const key = viewMode === 'all' ? `${i.date.getFullYear()}-${m}` : m;
        
        if (!evolutionStats[key]) evolutionStats[key] = { spent: 0, saving: 0 };
        evolutionStats[key].spent += (i.preco_total || 0);
        evolutionStats[key].saving += (i.saving_reais || 0);
    });
    
    const evolutionChartData = Object.entries(evolutionStats).map(([key, val]) => ({
        name: viewMode === 'year' ? `${key}` : key,
        spent: val.spent,
        saving: val.saving
    })); // Sort if needed

    return {
        uniquePurchases,
        totalSpent,
        totalSaving,
        uniqueSuppliers,
        uniqueMaterials,
        buyerChartData,
        supplierChartData,
        evolutionChartData
    };

  }, [items, purchases, viewMode, selectedYear, selectedMonth]);

  return (
    <div className="space-y-8 max-w-7xl mx-auto">
      <div className="flex flex-col md:flex-row md:items-center justify-between gap-4">
        <div>
          <h1 className="text-3xl font-bold text-slate-900">Dashboard Executivo</h1>
          <p className="text-slate-500 mt-1">Indicadores de performance e volume de compras</p>
        </div>
        
        <div className="flex gap-2 bg-white p-2 rounded-lg shadow-sm border">
            <Select value={viewMode} onValueChange={setViewMode}>
                <SelectTrigger className="w-[140px]"><SelectValue /></SelectTrigger>
                <SelectContent>
                    <SelectItem value="month">Mês Selecionado</SelectItem>
                    <SelectItem value="year">Ano Selecionado</SelectItem>
                    <SelectItem value="all">Todo o Histórico</SelectItem>
                </SelectContent>
            </Select>

            {(viewMode === 'month' || viewMode === 'year') && (
                <Select value={selectedYear} onValueChange={setSelectedYear}>
                    <SelectTrigger className="w-[100px]"><SelectValue /></SelectTrigger>
                    <SelectContent>
                        <SelectItem value="2023">2023</SelectItem>
                        <SelectItem value="2024">2024</SelectItem>
                        <SelectItem value="2025">2025</SelectItem>
                    </SelectContent>
                </Select>
            )}

            {viewMode === 'month' && (
                <Select value={selectedMonth} onValueChange={setSelectedMonth}>
                    <SelectTrigger className="w-[120px]"><SelectValue /></SelectTrigger>
                    <SelectContent>
                        {Array.from({length: 12}, (_, i) => i + 1).map(m => (
                            <SelectItem key={m} value={m.toString()}>{new Date(0, m-1).toLocaleString('pt-BR', {month: 'long'})}</SelectItem>
                        ))}
                    </SelectContent>
                </Select>
            )}
        </div>
      </div>

      {!stats ? <div className="p-8 text-center">Carregando dados...</div> : (
        <>
            <div className="grid gap-6 md:grid-cols-2 lg:grid-cols-5">
                <StatCard title="Compras" value={stats.uniquePurchases} icon={Package} color="bg-blue-500" subtext="Qtd. Pedidos" />
                <StatCard title="Gasto Total" value={`R$ ${(stats.totalSpent/1000).toFixed(1)}k`} icon={DollarSign} color="bg-emerald-600" subtext="Volume Financeiro" />
                <StatCard title="Saving" value={`R$ ${(stats.totalSaving/1000).toFixed(1)}k`} icon={TrendingUp} color="bg-green-500" subtext="Economia Gerada" />
                <StatCard title="Fornecedores" value={stats.uniqueSuppliers} icon={Users} color="bg-purple-500" subtext="Ativos no período" />
                <StatCard title="Itens" value={stats.uniqueMaterials} icon={BarChart3} color="bg-orange-500" subtext="Produtos distintos" />
            </div>

            <div className="grid gap-6 md:grid-cols-2">
                <Card>
                    <CardHeader><CardTitle>Gasto por Comprador</CardTitle></CardHeader>
                    <CardContent className="h-[300px]">
                        <ResponsiveContainer width="100%" height="100%">
                            <BarChart data={stats.buyerChartData} layout="vertical" margin={{left: 40}}>
                                <CartesianGrid strokeDasharray="3 3" horizontal={true} vertical={false} />
                                <XAxis type="number" hide />
                                <YAxis dataKey="name" type="category" width={100} tick={{fontSize: 12}} />
                                <Tooltip formatter={(val) => `R$ ${val.toLocaleString('pt-BR')}`} />
                                <Bar dataKey="value" fill="#3b82f6" radius={[0,4,4,0]} barSize={20} />
                            </BarChart>
                        </ResponsiveContainer>
                    </CardContent>
                </Card>

                <Card>
                    <CardHeader><CardTitle>Top Fornecedores</CardTitle></CardHeader>
                    <CardContent className="h-[300px]">
                        <ResponsiveContainer width="100%" height="100%">
                            <BarChart data={stats.supplierChartData} layout="vertical" margin={{left: 40}}>
                                <CartesianGrid strokeDasharray="3 3" horizontal={true} vertical={false} />
                                <XAxis type="number" hide />
                                <YAxis dataKey="name" type="category" width={100} tick={{fontSize: 12}} />
                                <Tooltip formatter={(val) => `R$ ${val.toLocaleString('pt-BR')}`} />
                                <Bar dataKey="value" fill="#8b5cf6" radius={[0,4,4,0]} barSize={20} />
                            </BarChart>
                        </ResponsiveContainer>
                    </CardContent>
                </Card>

                <Card className="md:col-span-2">
                    <CardHeader><CardTitle>Evolução: Gasto vs Saving</CardTitle></CardHeader>
                    <CardContent className="h-[300px]">
                        <ResponsiveContainer width="100%" height="100%">
                            <LineChart data={stats.evolutionChartData}>
                                <CartesianGrid strokeDasharray="3 3" vertical={false} />
                                <XAxis dataKey="name" />
                                <YAxis />
                                <Tooltip formatter={(val) => `R$ ${val.toLocaleString('pt-BR')}`} />
                                <Line type="monotone" dataKey="spent" name="Gasto Total" stroke="#3b82f6" strokeWidth={2} dot={{r:4}} />
                                <Line type="monotone" dataKey="saving" name="Saving" stroke="#10b981" strokeWidth={2} dot={{r:4}} />
                            </LineChart>
                        </ResponsiveContainer>
                    </CardContent>
                </Card>
            </div>
        </>
      )}
    </div>
  );
}
