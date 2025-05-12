# inv
invoices 
import React, { useState, useEffect } from 'react';
import { View, ScrollView, StyleSheet, TouchableOpacity, Text, FlatList, Image } from 'react-native';
import { TextInput, Button, Card, Title, Divider, IconButton, List } from 'react-native-paper';
import * as ImagePicker from 'expo-image-picker';
import * as FileSystem from 'expo-file-system';
import * as Sharing from 'expo-sharing';
import { Print } from 'expo-print';
import * as SQLite from 'expo-sqlite';
import { NavigationContainer } from '@react-navigation/native';
import { createStackNavigator } from '@react-navigation/stack';

// Initialize database
const db = SQLite.openDatabase('invoices.db');

// Initialize database tables
const initDatabase = () => {
  db.transaction(tx => {
    tx.executeSql(
      `CREATE TABLE IF NOT EXISTS documents (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        type TEXT,
        number TEXT,
        client_name TEXT,
        client_email TEXT,
        client_phone TEXT,
        client_address TEXT,
        subtotal REAL,
        tax REAL,
        total REAL,
        notes TEXT,
        created_at TEXT,
        status TEXT
      );`
    );
    
    tx.executeSql(
      `CREATE TABLE IF NOT EXISTS items (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        document_id INTEGER,
        description TEXT,
        quantity INTEGER,
        price REAL,
        tax_rate REAL,
        image TEXT,
        FOREIGN KEY(document_id) REFERENCES documents(id)
      );`
    );
    
    tx.executeSql(
      `CREATE TABLE IF NOT EXISTS company (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT,
        logo TEXT,
        address TEXT,
        phone TEXT,
        email TEXT,
        tax_id TEXT,
        bank_info TEXT
      );`
    );
  });
};

// Home Screen
const HomeScreen = ({ navigation }) => {
  return (
    <View style={styles.container}>
      <Card style={styles.card}>
        <Card.Content>
          <Title style={styles.title}>Invoice & Quote Generator</Title>
          <Button 
            mode="contained" 
            onPress={() => navigation.navigate('NewDocument')}
            style={styles.button}
            icon="file-document"
          >
            Create New Document
          </Button>
          <Button 
            mode="outlined" 
            onPress={() => navigation.navigate('DocumentList')}
            style={styles.button}
            icon="format-list-bulleted"
          >
            View All Documents
          </Button>
          <Button 
            mode="outlined" 
            onPress={() => navigation.navigate('CompanyProfile')}
            style={styles.button}
            icon="office-building"
          >
            Company Profile
          </Button>
        </Card.Content>
      </Card>
    </View>
  );
};

// New Document Screen
const NewDocumentScreen = ({ navigation }) => {
  const [documentType, setDocumentType] = useState('invoice');
  const [items, setItems] = useState([]);
  const [clientInfo, setClientInfo] = useState({
    name: '',
    email: '',
    phone: '',
    address: '',
  });
  const [companyInfo, setCompanyInfo] = useState({
    name: 'My Company',
    logo: null,
    address: '123 Business St, City',
    phone: '(123) 456-7890',
    email: 'contact@mycompany.com',
  });
  const [notes, setNotes] = useState('Thank you for your business!');

  // Load company info
  useEffect(() => {
    db.transaction(tx => {
      tx.executeSql(
        'SELECT * FROM company LIMIT 1',
        [],
        (_, { rows }) => {
          if (rows._array.length > 0) {
            setCompanyInfo(rows._array[0]);
          }
        }
      );
    });
  }, []);

  const addItem = () => {
    setItems([...items, {
      id: Date.now().toString(),
      description: '',
      quantity: 1,
      price: 0,
      taxRate: 10,
      image: null,
    }]);
  };

  const updateItem = (id, field, value) => {
    setItems(items.map(item => 
      item.id === id ? { ...item, [field]: value } : item
    ));
  };

  const removeItem = (id) => {
    setItems(items.filter(item => item.id !== id));
  };

  const addImage = async (itemId) => {
    let result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsEditing: true,
      aspect: [4, 3],
      quality: 1,
    });

    if (!result.canceled) {
      updateItem(itemId, 'image', result.assets[0].uri);
    }
  };

  const calculateTotals = () => {
    const subtotal = items.reduce((sum, item) => sum + (item.quantity * item.price), 0);
    const tax = items.reduce((sum, item) => sum + (item.quantity * item.price * item.taxRate / 100), 0);
    const total = subtotal + tax;
    return { subtotal, tax, total };
  };

  const { subtotal, tax, total } = calculateTotals();

  const generatePDF = async () => {
    const html = `
      <html>
        <head>
          <style>
            body { font-family: Arial; padding: 20px; }
            .header { display: flex; justify-content: space-between; margin-bottom: 20px; }
            .logo { max-width: 150px; max-height: 100px; }
            .document-title { font-size: 24px; font-weight: bold; text-align: center; margin: 20px 0; }
            .client-info, .company-info { margin-bottom: 20px; }
            table { width: 100%; border-collapse: collapse; margin: 20px 0; }
            th { text-align: left; padding: 8px; background: #f2f2f2; }
            td { padding: 8px; border-bottom: 1px solid #ddd; }
            .totals { margin-left: auto; width: 300px; }
            .notes { margin-top: 30px; }
          </style>
        </head>
        <body>
          <div class="header">
            <div class="company-info">
              <h2>${companyInfo.name}</h2>
              <p>${companyInfo.address}</p>
              <p>Phone: ${companyInfo.phone}</p>
              <p>Email: ${companyInfo.email}</p>
            </div>
            ${companyInfo.logo ? `<img class="logo" src="${companyInfo.logo}" />` : ''}
          </div>

          <div class="document-title">
            ${documentType === 'invoice' ? 'INVOICE' : 'QUOTE'} #${Math.floor(Math.random() * 10000)}
          </div>

          <div class="client-info">
            <h3>Bill To:</h3>
            <p>${clientInfo.name}</p>
            <p>${clientInfo.address}</p>
            <p>Phone: ${clientInfo.phone}</p>
            <p>Email: ${clientInfo.email}</p>
          </div>

          <table>
            <thead>
              <tr>
                <th>Description</th>
                <th>Qty</th>
                <th>Price</th>
                <th>Tax</th>
                <th>Amount</th>
              </tr>
            </thead>
            <tbody>
              ${items.map(item => `
                <tr>
                  <td>${item.description}</td>
                  <td>${item.quantity}</td>
                  <td>$${item.price.toFixed(2)}</td>
                  <td>${item.taxRate}%</td>
                  <td>$${(item.quantity * item.price * (1 + item.taxRate/100)).toFixed(2)}</td>
                </tr>
              `).join('')}
            </tbody>
          </table>

          <div class="totals">
            <p>Subtotal: $${subtotal.toFixed(2)}</p>
            <p>Tax: $${tax.toFixed(2)}</p>
            <p><strong>Total: $${total.toFixed(2)}</strong></p>
          </div>

          <div class="notes">
            <p><strong>Notes:</strong></p>
            <p>${notes}</p>
          </div>
        </body>
      </html>
    `;

    const { uri } = await Print.printToFileAsync({ html });
    return uri;
  };

  const saveDocument = async () => {
    const docNumber = `INV-${Math.floor(Math.random() * 10000)}`;
    const pdfUri = await generatePDF();
    
    db.transaction(tx => {
      tx.executeSql(
        `INSERT INTO documents 
        (type, number, client_name, client_email, client_phone, client_address, 
         subtotal, tax, total, notes, created_at, status) 
        VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, datetime('now'), 'draft')`,
        [
          documentType,
          docNumber,
          clientInfo.name,
          clientInfo.email,
          clientInfo.phone,
          clientInfo.address,
          subtotal,
          tax,
          total,
          notes,
        ],
        (_, result) => {
          const documentId = result.insertId;
          items.forEach(item => {
            tx.executeSql(
              `INSERT INTO items 
              (document_id, description, quantity, price, tax_rate, image) 
              VALUES (?, ?, ?, ?, ?, ?)`,
              [
                documentId,
                item.description,
                item.quantity,
                item.price,
                item.taxRate,
                item.image,
              ]
            );
          });
          alert('Document saved successfully!');
          navigation.navigate('DocumentList');
        },
        (_, error) => alert('Error saving document: ' + error.message)
      );
    });
  };

  const sharePDF = async () => {
    try {
      const pdfUri = await generatePDF();
      await Sharing.shareAsync(pdfUri, {
        dialogTitle: 'Share Document',
        mimeType: 'application/pdf',
        UTI: 'com.adobe.pdf'
      });
    } catch (error) {
      alert('Error sharing document: ' + error.message);
    }
  };

  return (
    <ScrollView style={styles.container}>
      <Card style={styles.card}>
        <Card.Content>
          <Title>New {documentType === 'invoice' ? 'Invoice' : 'Quote'}</Title>
          <Button 
            mode="outlined" 
            onPress={() => setDocumentType(documentType === 'invoice' ? 'quote' : 'invoice')}
            icon={documentType === 'invoice' ? 'format-quote-close' : 'receipt'}
          >
            Switch to {documentType === 'invoice' ? 'Quote' : 'Invoice'}
          </Button>
        </Card.Content>
      </Card>

      <Card style={styles.card}>
        <Card.Content>
          <Title>Client Information</Title>
          <TextInput
            label="Client Name"
            value={clientInfo.name}
            onChangeText={text => setClientInfo({...clientInfo, name: text})}
            style={styles.input}
          />
          <TextInput
            label="Email"
            value={clientInfo.email}
            onChangeText={text => setClientInfo({...clientInfo, email: text})}
            style={styles.input}
            keyboardType="email-address"
          />
          <TextInput
            label="Phone"
            value={clientInfo.phone}
            onChangeText={text => setClientInfo({...clientInfo, phone: text})}
            style={styles.input}
            keyboardType="phone-pad"
          />
          <TextInput
            label="Address"
            value={clientInfo.address}
            onChangeText={text => setClientInfo({...clientInfo, address: text})}
            style={styles.input}
            multiline
          />
        </Card.Content>
      </Card>

      <Card style={styles.card}>
        <Card.Content>
          <Title>Items</Title>
          {items.map((item) => (
            <View key={item.id} style={styles.itemContainer}>
              <TextInput
                label="Description"
                value={item.description}
                onChangeText={text => updateItem(item.id, 'description', text)}
                style={styles.input}
              />
              <View style={styles.row}>
                <TextInput
                  label="Qty"
                  value={String(item.quantity)}
                  onChangeText={text => updateItem(item.id, 'quantity', Number(text) || 0)}
                  style={[styles.input, styles.smallInput]}
                  keyboardType="numeric"
                />
                <TextInput
                  label="Price"
                  value={String(item.price)}
                  onChangeText={text => updateItem(item.id, 'price', Number(text) || 0)}
                  style={[styles.input, styles.smallInput]}
                  keyboardType="numeric"
                />
                <TextInput
                  label="Tax %"
                  value={String(item.taxRate)}
                  onChangeText={text => updateItem(item.id, 'taxRate', Number(text) || 0)}
                  style={[styles.input, styles.smallInput]}
                  keyboardType="numeric"
                />
              </View>
              <View style={styles.row}>
                <Button 
                  mode="outlined" 
                  onPress={() => addImage(item.id)}
                  style={styles.button}
                  icon="image"
                >
                  Add Image
                </Button>
                <IconButton
                  icon="delete"
                  size={20}
                  onPress={() => removeItem(item.id)}
                  style={styles.deleteButton}
                />
              </View>
              {item.image && (
                <Image 
                  source={{ uri: item.image }} 
                  style={styles.itemImage} 
                />
              )}
              <Divider style={styles.divider} />
            </View>
          ))}
          <Button mode="contained" onPress={addItem} style={styles.addButton} icon="plus">
            Add Item
          </Button>
        </Card.Content>
      </Card>

      <Card style={styles.card}>
        <Card.Content>
          <Title>Notes</Title>
          <TextInput
            label="Notes/Terms"
            value={notes}
            onChangeText={setNotes}
            style={styles.input}
            multiline
          />
        </Card.Content>
      </Card>

      <Card style={styles.card}>
        <Card.Content>
          <Title>Totals</Title>
          <Text>Subtotal: ${subtotal.toFixed(2)}</Text>
          <Text>Tax: ${tax.toFixed(2)}</Text>
          <Text style={styles.totalText}>Total: ${total.toFixed(2)}</Text>
        </Card.Content>
      </Card>

      <View style={styles.actionButtons}>
        <Button 
          mode="contained" 
          onPress={saveDocument}
          style={styles.actionButton}
          icon="content-save"
        >
          Save Document
        </Button>
        <Button 
          mode="contained" 
          onPress={sharePDF}
          style={styles.actionButton}
          icon="share"
        >
          Share as PDF
        </Button>
      </View>
    </ScrollView>
  );
};

// Company Profile Screen
const CompanyProfileScreen = ({ navigation }) => {
  const [companyInfo, setCompanyInfo] = useState({
    name: 'My Company',
    logo: null,
    address: '123 Business St, City',
    phone: '(123) 456-7890',
    email: 'contact@mycompany.com',
    taxId: '',
    bankInfo: '',
  });

  useEffect(() => {
    db.transaction(tx => {
      tx.executeSql(
        'SELECT * FROM company LIMIT 1',
        [],
        (_, { rows }) => {
          if (rows._array.length > 0) {
            setCompanyInfo(rows._array[0]);
          }
        }
      );
    });
  }, []);

  const addLogo = async () => {
    let result = await ImagePicker.launchImageLibraryAsync({
      mediaTypes: ImagePicker.MediaTypeOptions.Images,
      allowsEditing: true,
      aspect: [4, 3],
      quality: 1,
    });

    if (!result.canceled) {
      setCompanyInfo({...companyInfo, logo: result.assets[0].uri});
    }
  };

  const saveProfile = () => {
    db.transaction(tx => {
      tx.executeSql(
        'DELETE FROM company',
        [],
        () => {
          tx.executeSql(
            `INSERT INTO company 
            (name, logo, address, phone, email, tax_id, bank_info) 
            VALUES (?, ?, ?, ?, ?, ?, ?)`,
            [
              companyInfo.name,
              companyInfo.logo,
              companyInfo.address,
              companyInfo.phone,
              companyInfo.email,
              companyInfo.taxId,
              companyInfo.bankInfo,
            ],
            () => {
              alert('Company profile saved!');
              navigation.goBack();
            },
            (_, error) => alert('Error saving profile: ' + error.message)
          );
        },
        (_, error) => alert('Error saving profile: ' + error.message)
      );
    });
  };

  return (
    <ScrollView style={styles.container}>
      <Card style={styles.card}>
        <Card.Content>
          <Title>Company Profile</Title>
          <Button 
            mode="outlined" 
            onPress={addLogo}
            style={styles.button}
            icon="image"
          >
            {companyInfo.logo ? 'Change Logo' : 'Add Logo'}
          </Button>
          
          {companyInfo.logo && (
            <Image 
              source={{ uri: companyInfo.logo }} 
              style={styles.logo} 
            />
          )}
          
          <TextInput
            label="Company Name"
            value={companyInfo.name}
            onChangeText={text => setCompanyInfo({...companyInfo, name: text})}
            style={styles.input}
          />
          <TextInput
            label="Address"
            value={companyInfo.address}
            onChangeText={text => setCompanyInfo({...companyInfo, address: text})}
            style={styles.input}
            multiline
          />
          <TextInput
            label="Phone"
            value={companyInfo.phone}
            onChangeText={text => setCompanyInfo({...companyInfo, phone: text})}
            style={styles.input}
            keyboardType="phone-pad"
          />
          <TextInput
            label="Email"
            value={companyInfo.email}
            onChangeText={text => setCompanyInfo({...companyInfo, email: text})}
            style={styles.input}
            keyboardType="email-address"
          />
          <TextInput
            label="Tax ID"
            value={companyInfo.taxId}
            onChangeText={text => setCompanyInfo({...companyInfo, taxId: text})}
            style={styles.input}
          />
          <TextInput
            label="Bank Information"
            value={companyInfo.bankInfo}
            onChangeText={text => setCompanyInfo({...companyInfo, bankInfo: text})}
            style={styles.input}
            multiline
          />
          
          <Button 
            mode="contained" 
            onPress={saveProfile}
            style={styles.saveButton}
            icon="content-save"
          >
            Save Profile
          </Button>
        </Card.Content>
      </Card>
    </ScrollView>
  );
};

// Document List Screen
const DocumentListScreen = ({ navigation }) => {
  const [documents, setDocuments] = useState([]);

  useEffect(() => {
    loadDocuments();
  }, []);

  const loadDocuments = () => {
    db.transaction(tx => {
      tx.executeSql(
        `SELECT documents.* FROM documents ORDER BY created_at DESC`,
        [],
        (_, { rows }) => setDocuments(rows._array),
        (_, error) => alert('Error loading documents: ' + error.message)
      );
    });
  };

  const viewDocument = (id) => {
    navigation.navigate('ViewDocument', { id });
  };

  return (
    <View style={styles.container}>
      <Card style={styles.card}>
        <Card.Content>
          <Title>Your Documents</Title>
          {documents.length === 0 ? (
            <Text style={styles.noDocuments}>No documents found</Text>
          ) : (
            <FlatList
              data={documents}
              keyExtractor={item => item.id.toString()}
              renderItem={({ item }) => (
                <List.Item
                  title={`${item.type === 'invoice' ? 'Invoice' : 'Quote'} #${item.number}`}
                  description={`${item.client_name} - $${item.total.toFixed(2)}`}
                  left={props => <List.Icon {...props} icon={item.type === 'invoice' ? 'receipt' : 'format-quote-close'} />}
                  onPress={() => viewDocument(item.id)}
                />
              )}
            />
          )}
          <Button 
            mode="contained" 
            onPress={() => navigation.navigate('NewDocument')}
            style={styles.button}
            icon="plus"
          >
            Create New
          </Button>
        </Card.Content>
      </Card>
    </View>
  );
};

// View Document Screen
const ViewDocumentScreen = ({ route, navigation }) => {
  const { id } = route.params;
  const [document, setDocument] = useState(null);
  const [items, setItems] = useState([]);
  const [companyInfo, setCompanyInfo] = useState(null);

  useEffect(() => {
    loadDocument();
    loadCompanyInfo();
  }, []);

  const loadDocument = () => {
    db.transaction(tx => {
      tx.executeSql(
        'SELECT * FROM documents WHERE id = ?',
        [id],
        (_, { rows }) => {
          const doc = rows._array[0];
          if (doc) {
            setDocument(doc);
            tx.executeSql(
              'SELECT * FROM items WHERE document_id = ?',
              [id],
              (_, { itemRows }) => setItems(itemRows._array)
            );
          }
        }
      );
    });
  };

  const loadCompanyInfo = () => {
    db.transaction(tx => {
      tx.executeSql(
        'SELECT * FROM company LIMIT 1',
        [],
        (_, { rows }) => {
          if (rows._array.length > 0) {
            setCompanyInfo(rows._array[0]);
          }
        }
      );
    });
  };

  const shareDocument = async () => {
    if (!document || !companyInfo) return;

    const html = `
      <html>
        <head>
          <style>
            body { font-family: Arial; padding: 20px; }
            .header { display: flex; justify-content: space-between; margin-bottom: 20px; }
            .logo { max-width: 150px; max-height: 100px; }
            .document-title { font-size: 24px; font-weight: bold; text-align: center; margin: 20px 0; }
            .client-info, .company-info { margin-bottom: 20px; }
            table { width: 100%; border-collapse: collapse; margin: 20px 0; }
            th { text-align: left; padding: 8px; background: #f2f2f2; }
            td { padding: 8px; border-bottom: 1px solid #ddd; }
            .totals { margin-left: auto; width: 300px; }
            .notes { margin-top: 30px; }
          </style>
        </head>
        <body>
          <div class="header">
            <div class="company-info">
              <h2>${companyInfo.name}</h2>
              <p>${companyInfo.address}</p>
              <p>Phone: ${companyInfo.phone}</p>
              <p>Email: ${companyInfo.email}</p>
            </div>
            ${companyInfo.logo ? `<img class="logo" src="${companyInfo.logo}" />` : ''}
          </div>

          <div class="document-title">
            ${document.type === 'invoice' ? 'INVOICE' : 'QUOTE'} #${document.number}
          </div>

          <div class="client-info">
            <h3>Bill To:</h3>
            <p>${document.client_name}</p>
            <p>${document.client_address}</p>
            <p>Phone: ${document.client_phone}</p>
            <p>Email: ${document.client_email}</p>
          </div>

          <table>
            <thead>
              <tr>
                <th>Description</th>
                <th>Qty</th>
                <th>Price</th>
                <th>Tax</th>
                <th>Amount</th>
              </tr>
            </thead>
            <tbody>
              ${items.map(item => `
                <tr>
                  <td>${item.description}</td>
                  <td>${item.quantity}</td>
                  <td>$${item.price.toFixed(2)}</td>
                  <td>${item.tax_rate}%</td>
                  <td>$${(item.quantity * item.price * (1 + item.tax_rate/100)).toFixed(2)}</td>
                </tr>
              `).join('')}
            </tbody>
          </table>

          <div class="totals">
            <p>Subtotal: $${document.subtotal.toFixed(2)}</p>
            <p>Tax: $${document.tax.toFixed(2)}</p>
            <p><strong>Total: $${document.total.toFixed(2)}</strong></p>
          </div>

          <div class="notes">
            <p><strong>Notes:</strong></p>
            <p>${document.notes}</p>
          </div>
        </body>
      </html>
    `;

    try {
      const { uri } = await Print.printToFileAsync({ html });
      await Sharing.shareAsync(uri, {
        dialogTitle: 'Share Document',
        mimeType: 'application/pdf',
        UTI: 'com.adobe.pdf'
      });
    } catch (error) {
      alert('Error sharing document: ' + error.message);
    }
  };

  if (!document) {
    return (
      <View style={styles.container}>
        <Text>Loading...</Text>
      </View>
    );
  }

  return (
    <ScrollView style={styles.container}>
      <Card style={styles.card}>
        <Card.Content>
          <Title>{document.type === 'invoice' ? 'Invoice' : 'Quote'} #{document.number}</Title>
          <Text>Date: {new Date(document.created_at).toLocaleDateString()}</Text>
          <Text>Status: {document.status}</Text>
        </Card.Content>
      </Card>

      <Card style={styles.card}>
        <Card.Content>
          <Title>Client Information</Title>
          <Text>Name: {document.client_name}</Text>
          <Text>Address: {document.client_address}</Text>
          <Text>Phone: {document.client_phone}</Text>
          <Text>Email: {document.client_email}</Text>
        </Card.Content>
      </Card>

      <Card style={styles.card}>
        <Card.Content>
          <Title>Items</Title>
          {items.map((item) => (
            <View key={item.id} style={styles.itemContainer}>
              <Text>Description: {item.description}</Text>
              <Text>Quantity: {item.quantity}</Text>
              <Text>Price: ${item.price.toFixed(2)}</Text>
              <Text>Tax: {item.tax_rate}%</Text>
              {item.image && (
                <Image 
                  source={{ uri: item.image }} 
                  style={styles.itemImage} 
                />
              )}
              <Divider style={styles.divider} />
            </View>
          ))}
        </Card.Content>
      </Card>

      <Card style={styles.card}>
        <Card.Content>
          <Title>Totals</Title>
          <Text>Subtotal: ${document.subtotal.toFixed(2)}</Text>
          <Text>Tax: ${document.tax.toFixed(2)}</Text>
          <Text style={styles.totalText}>Total: ${document.total.toFixed(2)}</Text>
        </Card.Content>
      </Card>

      <Card style={styles.card}>
        <Card.Content>
          <Title>Notes</Title>
          <Text>{document.notes}</Text>
        </Card.Content>
      </Card>

      <View style={styles.actionButtons}>
        <Button 
          mode="contained" 
          onPress={shareDocument}
          style={styles.actionButton}
          icon="share"
        >
          Share as PDF
        </Button>
      </View>
    </ScrollView>
  );
};

// Main App Component
const Stack = createStackNavigator();

const App = () => {
  useEffect(() => {
    initDatabase();
  }, []);

  return (
    <NavigationContainer>
      <Stack.Navigator initialRouteName="Home">
        <Stack.Screen name="Home" component={HomeScreen} options={{ title: 'Invoice App' }} />
        <Stack.Screen name="NewDocument" component={NewDocumentScreen} options={{ title: 'New Document' }} />
        <Stack.Screen name="DocumentList" component={DocumentListScreen} options={{ title: 'Your Documents' }} />
        <Stack.Screen name="CompanyProfile" component={CompanyProfileScreen} options={{ title: 'Company Profile' }} />
        <Stack.Screen name="ViewDocument" component={ViewDocumentScreen} options={{ title: 'Document Details' }} />
      </Stack.Navigator>
    </NavigationContainer>
  );
};

// Styles
const styles = StyleSheet.create({
  container: {
    flex: 1,
    padding: 10,
  },
  card: {
    marginBottom: 15,
  },
  title: {
    textAlign: 'center',
    marginBottom: 20,
  },
  input: {
    marginBottom: 10,
  },
  smallInput: {
    flex: 1,
    marginRight: 10,
  },
  row: {
    flexDirection: 'row',
    alignItems: 'center',
  },
  itemContainer: {
    marginBottom: 15,
  },
  itemImage: {
    width: 100,
    height: 100,
    marginTop: 10,
  },
  divider: {
    marginVertical: 10,
  },
  addButton: {
    marginTop: 10,
  },
  deleteButton: {
    marginLeft: 10,
  },
  button: {
    marginVertical: 10,
  },
  totalText: {
    fontSize: 18,
    fontWeight: 'bold',
    marginTop: 5,
  },
  actionButtons: {
    marginVertical: 20,
  },
  actionButton: {
    marginBottom: 10,
  },
  logo: {
    width: 150,
    height: 100,
    marginBottom: 15,
    alignSelf: 'center',
  },
  saveButton: {
    marginTop: 20,
  },
  noDocuments: {
    textAlign: 'center',
    marginVertical: 20,
  },
});

export default App;
