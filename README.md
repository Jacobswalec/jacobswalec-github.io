# jacobswalec-github.io
finance tracker

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta name="description" content="Track your personal finances with income, expenses, and visual charts">
    <title>Personal Finance Tracker</title>
    
    <!-- React -->
    <script crossorigin src="https://cdnjs.cloudflare.com/ajax/libs/react/18.2.0/umd/react.production.min.js"></script>
    <script crossorigin src="https://cdnjs.cloudflare.com/ajax/libs/react-dom/18.2.0/umd/react-dom.production.min.js"></script>
    
    <!-- Babel for JSX -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/babel-standalone/7.23.5/babel.min.js"></script>
    
    <!-- Recharts -->
    <script src="https://cdnjs.cloudflare.com/ajax/libs/recharts/2.10.3/Recharts.js"></script>
    
    <!-- Tailwind CSS -->
    <script src="https://cdn.tailwindcss.com"></script>
    
    <style>
        body {
            margin: 0;
            padding: 0;
            font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', 'Roboto', 'Oxygen', 'Ubuntu', 'Cantarell', sans-serif;
        }
        
        /* Prevent flash of unstyled content */
        #root:empty::before {
            content: "Loading...";
            display: flex;
            justify-content: center;
            align-items: center;
            min-height: 100vh;
            font-size: 1.5rem;
            color: #6366f1;
        }
    </style>
</head>
<body>
    <div id="root"></div>
    
    <script type="text/babel">
        const { useState, useEffect } = React;
        const { BarChart, Bar, XAxis, YAxis, Tooltip, ResponsiveContainer, Cell } = Recharts;

        // localStorage wrapper to match storage API
        window.storage = {
            async get(key) {
                const value = localStorage.getItem(key);
                return value ? { key, value } : null;
            },
            async set(key, value) {
                localStorage.setItem(key, value);
                return { key, value };
            },
            async delete(key) {
                localStorage.removeItem(key);
                return { key, deleted: true };
            },
            async list(prefix = '') {
                const keys = Object.keys(localStorage).filter(k => k.startsWith(prefix));
                return { keys };
            }
        };

        // Icons as SVG components
        const DollarIcon = () => (
            <svg width="40" height="40" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
                <line x1="12" y1="1" x2="12" y2="23"></line>
                <path d="M17 5H9.5a3.5 3.5 0 0 0 0 7h5a3.5 3.5 0 0 1 0 7H6"></path>
            </svg>
        );

        const TrendingUpIcon = () => (
            <svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
                <polyline points="23 6 13.5 15.5 8.5 10.5 1 18"></polyline>
                <polyline points="17 6 23 6 23 12"></polyline>
            </svg>
        );

        const TrendingDownIcon = () => (
            <svg width="32" height="32" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
                <polyline points="23 18 13.5 8.5 8.5 13.5 1 6"></polyline>
                <polyline points="17 18 23 18 23 12"></polyline>
            </svg>
        );

        const ChartIcon = () => (
            <svg width="20" height="20" viewBox="0 0 24 24" fill="none" stroke="currentColor" strokeWidth="2">
                <path d="M21.21 15.89A10 10 0 1 1 8 2.83"></path>
                <path d="M22 12A10 10 0 0 0 12 2v10z"></path>
            </svg>
        );

        function FinanceTracker() {
            const [transactions, setTransactions] = useState([]);
            const [category, setCategory] = useState('');
            const [amount, setAmount] = useState('');
            const [description, setDescription] = useState('');
            const [showChart, setShowChart] = useState(false);

            useEffect(() => {
                loadTransactions();
            }, []);

            const loadTransactions = async () => {
                try {
                    const result = await window.storage.list('transaction:');
                    if (result && result.keys) {
                        const loadedTransactions = [];
                        for (const key of result.keys) {
                            const data = await window.storage.get(key);
                            if (data) {
                                loadedTransactions.push(JSON.parse(data.value));
                            }
                        }
                        loadedTransactions.sort((a, b) => new Date(b.date) - new Date(a.date));
                        setTransactions(loadedTransactions);
                    }
                } catch (error) {
                    console.log('No transactions yet');
                }
            };

            const addTransaction = async (type) => {
                if (!category.trim()) {
                    alert('Please enter a category.');
                    return;
                }
                
                const amountNum = parseFloat(amount);
                if (isNaN(amountNum) || amountNum <= 0) {
                    alert('Please enter a valid amount.');
                    return;
                }

                const transaction = {
                    id: Date.now(),
                    date: new Date().toISOString(),
                    type,
                    category: category.trim(),
                    amount: amountNum,
                    description: description.trim()
                };

                try {
                    await window.storage.set(`transaction:${transaction.id}`, JSON.stringify(transaction));
                    setCategory('');
                    setAmount('');
                    setDescription('');
                    await loadTransactions();
                } catch (error) {
                    alert('Error saving transaction');
                }
            };

            const deleteTransaction = async (id) => {
                if (window.confirm('Are you sure you want to delete this transaction?')) {
                    try {
                        await window.storage.delete(`transaction:${id}`);
                        await loadTransactions();
                    } catch (error) {
                        alert('Error deleting transaction');
                    }
                }
            };

            const clearAllData = async () => {
                if (window.confirm('⚠️ This will delete ALL transactions. Are you sure?')) {
                    try {
                        const result = await window.storage.list('transaction:');
                        if (result && result.keys) {
                            for (const key of result.keys) {
                                await window.storage.delete(key);
                            }
                        }
                        await loadTransactions();
                        alert('All data cleared successfully!');
                    } catch (error) {
                        alert('Error clearing data');
                    }
                }
            };

            const calculateSummary = () => {
                const income = transactions
                    .filter(t => t.type === 'income')
                    .reduce((sum, t) => sum + t.amount, 0);
                
                const expense = transactions
                    .filter(t => t.type === 'expense')
                    .reduce((sum, t) => sum + t.amount, 0);
                
                return { income, expense, balance: income - expense };
            };

            const getExpensesByCategory = () => {
                const expenses = {};
                transactions
                    .filter(t => t.type === 'expense')
                    .forEach(t => {
                        expenses[t.category] = (expenses[t.category] || 0) + t.amount;
                    });
                
                return Object.entries(expenses).map(([category, amount]) => ({
                    category,
                    amount
                }));
            };

            const formatDate = (dateString) => {
                const date = new Date(dateString);
                return date.toLocaleDateString() + ' ' + date.toLocaleTimeString([], { 
                    hour: '2-digit', 
                    minute: '2-digit' 
                });
            };

            const { income, expense, balance } = calculateSummary();
            const chartData = getExpensesByCategory();
            const colors = ['#ef4444', '#f97316', '#f59e0b', '#eab308', '#84cc16', '#22c55e', '#10b981', '#14b8a6', '#06b6d4', '#0ea5e9'];

            return (
                <div className="min-h-screen bg-gradient-to-br from-blue-50 to-indigo-100 p-4 md:p-6">
                    <div className="max-w-6xl mx-auto">
                        <div className="bg-white rounded-2xl shadow-xl p-4 md:p-8 mb-6">
                            <div className="flex justify-between items-start mb-8">
                                <h1 className="text-3xl md:text-4xl font-bold text-gray-800 flex items-center gap-3">
                                    <span className="text-green-600"><DollarIcon /></span>
                                    Personal Finance Tracker
                                </h1>
                                {transactions.length > 0 && (
                                    <button
                                        onClick={clearAllData}
                                        className="text-sm text-red-600 hover:text-red-800 font-medium px-3 py-1 border border-red-300 rounded-lg hover:bg-red-50 transition-colors"
                                    >
                                        Clear All
                                    </button>
                                )}
                            </div>

                            {/* Summary Cards */}
                            <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-8">
                                <div className="bg-gradient-to-br from-green-400 to-green-600 rounded-xl p-6 text-white shadow-lg">
                                    <div className="flex items-center justify-between">
                                        <div>
                                            <p className="text-green-100 text-sm font-medium">Total Income</p>
                                            <p className="text-2xl md:text-3xl font-bold mt-1">${income.toFixed(2)}</p>
                                        </div>
                                        <div className="opacity-80"><TrendingUpIcon /></div>
                                    </div>
                                </div>
                                
                                <div className="bg-gradient-to-br from-red-400 to-red-600 rounded-xl p-6 text-white shadow-lg">
                                    <div className="flex items-center justify-between">
                                        <div>
                                            <p className="text-red-100 text-sm font-medium">Total Expenses</p>
                                            <p className="text-2xl md:text-3xl font-bold mt-1">${expense.toFixed(2)}</p>
                                        </div>
                                        <div className="opacity-80"><TrendingDownIcon /></div>
                                    </div>
                                </div>
                                
                                <div className={`bg-gradient-to-br ${balance >= 0 ? 'from-blue-400 to-blue-600' : 'from-orange-400 to-orange-600'} rounded-xl p-6 text-white shadow-lg`}>
                                    <div className="flex items-center justify-between">
                                        <div>
                                            <p className="text-blue-100 text-sm font-medium">Balance</p>
                                            <p className="text-2xl md:text-3xl font-bold mt-1">${balance.toFixed(2)}</p>
                                        </div>
                                        <div className="opacity-80"><DollarIcon /></div>
                                    </div>
                                </div>
                            </div>

                            {/* Input Form */}
                            <div className="bg-gray-50 rounded-xl p-4 md:p-6 mb-6">
                                <div className="grid grid-cols-1 md:grid-cols-3 gap-4 mb-4">
                                    <div>
                                        <label className="block text-sm font-medium text-gray-700 mb-2">Category</label>
                                        <input
                                            type="text"
                                            value={category}
                                            onChange={(e) => setCategory(e.target.value)}
                                            placeholder="e.g., Groceries, Salary"
                                            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                                        />
                                    </div>
                                    
                                    <div>
                                        <label className="block text-sm font-medium text-gray-700 mb-2">Amount</label>
                                        <input
                                            type="number"
                                            value={amount}
                                            onChange={(e) => setAmount(e.target.value)}
                                            placeholder="0.00"
                                            step="0.01"
                                            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                                        />
                                    </div>
                                    
                                    <div>
                                        <label className="block text-sm font-medium text-gray-700 mb-2">Description</label>
                                        <input
                                            type="text"
                                            value={description}
                                            onChange={(e) => setDescription(e.target.value)}
                                            placeholder="Optional notes"
                                            className="w-full px-4 py-2 border border-gray-300 rounded-lg focus:ring-2 focus:ring-blue-500 focus:border-transparent"
                                        />
                                    </div>
                                </div>
                                
                                <div className="flex gap-3 flex-wrap">
                                    <button
                                        onClick={() => addTransaction('income')}
                                        className="flex-1 min-w-[120px] bg-green-500 hover:bg-green-600 text-white font-semibold py-3 px-4 md:px-6 rounded-lg transition-colors shadow-md"
                                    >
                                        Add Income
                                    </button>
                                    <button
                                        onClick={() => addTransaction('expense')}
                                        className="flex-1 min-w-[120px] bg-red-500 hover:bg-red-600 text-white font-semibold py-3 px-4 md:px-6 rounded-lg transition-colors shadow-md"
                                    >
                                        Add Expense
                                    </button>
                                    <button
                                        onClick={() => setShowChart(!showChart)}
                                        className="flex-1 min-w-[120px] bg-blue-500 hover:bg-blue-600 text-white font-semibold py-3 px-4 md:px-6 rounded-lg transition-colors shadow-md flex items-center justify-center gap-2"
                                    >
                                        <ChartIcon />
                                        {showChart ? 'Hide Chart' : 'Show Chart'}
                                    </button>
                                </div>
                            </div>

                            {/* Chart */}
                            {showChart && chartData.length > 0 && (
                                <div className="bg-gray-50 rounded-xl p-4 md:p-6 mb-6">
                                    <h2 className="text-xl font-bold text-gray-800 mb-4">Expenses by Category</h2>
                                    <ResponsiveContainer width="100%" height={300}>
                                        <BarChart data={chartData}>
                                            <XAxis dataKey="category" angle={-45} textAnchor="end" height={100} />
                                            <YAxis />
                                            <Tooltip formatter={(value) => `$${value.toFixed(2)}`} />
                                            <Bar dataKey="amount">
                                                {chartData.map((entry, index) => (
                                                    <Cell key={`cell-${index}`} fill={colors[index % colors.length]} />
                                                ))}
                                            </Bar>
                                        </BarChart>
                                    </ResponsiveContainer>
                                </div>
                            )}

                            {showChart && chartData.length === 0 && (
                                <div className="bg-gray-50 rounded-xl p-6 mb-6 text-center text-gray-500">
                                    No expense data to visualize yet. Add some expenses to see the chart!
                                </div>
                            )}

                            {/* Transactions Table */}
                            <div className="bg-white rounded-xl border border-gray-200 overflow-hidden">
                                <div className="overflow-x-auto">
                                    <table className="w-full">
                                        <thead className="bg-gray-100">
                                            <tr>
                                                <th className="px-4 py-3 text-left text-xs md:text-sm font-semibold text-gray-700">Date</th>
                                                <th className="px-4 py-3 text-left text-xs md:text-sm font-semibold text-gray-700">Type</th>
                                                <th className="px-4 py-3 text-left text-xs md:text-sm font-semibold text-gray-700">Category</th>
                                                <th className="px-4 py-3 text-left text-xs md:text-sm font-semibold text-gray-700">Amount</th>
                                                <th className="px-4 py-3 text-left text-xs md:text-sm font-semibold text-gray-700 hidden md:table-cell">Description</th>
                                                <th className="px-4 py-3 text-left text-xs md:text-sm font-semibold text-gray-700">Action</th>
                                            </tr>
                                        </thead>
                                        <tbody className="divide-y divide-gray-200">
                                            {transactions.length === 0 ? (
                                                <tr>
                                                    <td colSpan="6" className="px-4 py-8 text-center text-gray-500">
                                                        No transactions yet. Add your first income or expense above!
                                                    </td>
                                                </tr>
                                            ) : (
                                                transactions.map((t) => (
                                                    <tr key={t.id} className="hover:bg-gray-50 transition-colors">
                                                        <td className="px-4 py-3 text-xs md:text-sm text-gray-700">{formatDate(t.date)}</td>
                                                        <td className="px-4 py-3 text-xs md:text-sm">
                                                            <span className={`px-2 md:px-3 py-1 rounded-full text-xs font-medium ${
                                                                t.type === 'income' 
                                                                    ? 'bg-green-100 text-green-800' 
                                                                    : 'bg-red-100 text-red-800'
                                                            }`}>
                                                                {t.type}
                                                            </span>
                                                        </td>
                                                        <td className="px-4 py-3 text-xs md:text-sm text-gray-700 font-medium">{t.category}</td>
                                                        <td className="px-4 py-3 text-xs md:text-sm font-semibold text-gray-900">
                                                            ${t.amount.toFixed(2)}
                                                        </td>
                                                        <td className="px-4 py-3 text-xs md:text-sm text-gray-600 hidden md:table-cell">{t.description || '-'}</td>
                                                        <td className="px-4 py-3 text-xs md:text-sm">
                                                            <button
                                                                onClick={() => deleteTransaction(t.id)}
                                                                className="text-red-600 hover:text-red-800 font-medium"
                                                            >
                                                                Delete
                                                            </button>
                                                        </td>
                                                    </tr>
                                                ))
                                            )}
                                        </tbody>
                                    </table>
                                </div>
                            </div>
                        </div>
                        
                        {/* Footer */}
                        <div className="text-center text-gray-600 text-sm pb-4">
                            <p>💰 Your financial data is stored locally in your browser</p>
                        </div>
                    </div>
                </div>
            );
        }

        // Render the app
        ReactDOM.render(<FinanceTracker />, document.getElementById('root'));
    </script>
</body>
</html>
